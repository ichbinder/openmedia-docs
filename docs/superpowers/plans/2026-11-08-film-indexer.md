# Film-Indexer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Der Film-Knopf lädt mit einem Klick die beste Version — aus eigener NZB oder von Treasure Maps — und bietet optional eine Auswahl-Liste.

**Architecture:** Das API bewertet eigene `NzbFile`s und Treasure-Maps-Treffer gemeinsam in einer reinen Funktion (`movie-ranking.ts`) und liefert die sortierte Liste über `GET /movies/candidates`. `POST /movies/requests` startet die gewählte Version über die bestehenden Wege (`ingestNzb`, `DownloadJob`) und legt eine `MovieRequest` an, die der bestehende Reconciler für das Weiterprobieren nutzt. Die Webseite ersetzt in `download-button.tsx` die Datenquelle und die Zustände.

**Tech Stack:** Express 5, Prisma 7, PostgreSQL, Vitest + supertest (echte Test-DB über `src/test/setup.ts`); Next.js 16.4, React, Radix-UI-Komponenten in `src/components/ui`, Vitest + Testing Library.

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-08-film-indexer-design.md`

**Repos/Branches:** `openmedia-api` und `openmedia-web` (beide unter `/Users/jschattka/git_private/cinescope/`), jeweils Branch `feature/film-indexer` von frischem `main`.

## Global Constraints

- Alle Texte für Nutzer auf Deutsch, mit echten Umlauten.
- Keine neuen Abhängigkeiten.
- `rankReleases` und das Serien-Verhalten bleiben unverändert; alle bestehenden Tests in `openmedia-api` grün.
- Limit: 400 NZB-Downloads/24 h, geteilt mit Serien, Reserve `BUDGET_RESERVE = 50`. Nur `downloadNzb` zählt, Suchen nicht.
- Höchstens 3 Versuche pro Anforderung (`MAX_ATTEMPTS = 3`).
- Ziel-Größenbereich über `getTargetBand()` (Standard 1,5–6 GB/h). Laufzeit aus TMDB, sonst `DEFAULT_MOVIE_RUNTIME_MINUTES = 110`.
- Indexer-Suche je Film 1 h Cache (`SEARCH_TTL_MS`).
- Passwort-Releases (`password === 2`) nie anzeigen.
- Kein automatischer Wechsel auf bessere Versionen. Selbst gewählte Versionen (`manual`) werden nie automatisch ersetzt.
- Nie ungefragt eine Version der Gruppe `foreign` oder `cam` laden.
- Migration verändert keine bestehenden Daten.

## Review Focus

1. ffprobe liefert keine oder nur `und`-Sprach-Tags → die Version darf **nicht** als „kein deutscher Ton" `broken` werden (Test in Task 4).
2. Eigene NZB ohne Sprach-Hinweis im Namen und ohne ffprobe-Daten → Gruppe `foreign`, Schild „Kein Deutsch", **nicht** „Nur Englisch" (der Parser setzt sonst stumm `en`) (Test in Task 2).
3. Zwei Nutzer klicken denselben Indexer-Film kurz nacheinander → beide `MovieRequest`s zeigen auf dieselbe `NzbFile`, und **beide** bekommen den Film in die `UserLibrary` (Test in Task 4).
4. Indexer nicht eingerichtet oder gestört → `GET /movies/candidates` liefert trotzdem die eigenen NZBs mit `indexer: "unavailable"`, kein 5xx (Test in Task 5).
5. Die geklickte Version ist inzwischen weg (Cache neu, Version `broken`) → 404 `candidate_gone`; die Webseite lädt die Liste neu und zeigt „Version nicht mehr verfügbar" (Tests in Task 5 und Task 6).

---

### Task 1: Gemeinsame Bausteine im API

**Files:**
- Modify: `openmedia-api/src/lib/treasure-maps.ts`
- Modify: `openmedia-api/src/lib/nzb-parser.ts:66-77`
- Modify: `openmedia-api/src/lib/episode-requests.ts` (nutzt die verschobenen Helfer)
- Modify: `openmedia-api/src/lib/tmdb.ts:11-19,67-75`
- Test: `openmedia-api/src/lib/treasure-maps.test.ts`, `openmedia-api/src/lib/nzb-parser.test.ts`

**Interfaces:**
- Produces:
  - `searchMovie(tmdbId: number): Promise<IndexerRelease[]>` in `treasure-maps.ts`
  - `BUDGET_RESERVE = 50` und `withBudgetLock<T>(fn: () => Promise<T>): Promise<T>` in `treasure-maps.ts` (verschoben aus `episode-requests.ts`, inkl. `ponytail:`-Kommentar)
  - `toLanguageCodes(names: string[] | null): string[]` in `nzb-parser.ts` (verschoben aus `episode-requests.ts`)
  - `TmdbMovieResult.runtime: number | null`

- [ ] **Step 1: Failing tests schreiben**

```ts
// treasure-maps.test.ts
it("searchMovie queries /movie by TMDB id with extended data", async () => {
  // fetch-Mock wie bei den searchEpisode-Tests
  await searchMovie(603);
  expect(fetchUrl()).toBe("https://treasure-maps.com/api/v1/movie?tmdbid=603&extended=true&limit=100");
});
it("searchMovie caches per film for one hour", async () => {
  await searchMovie(603); await searchMovie(603);
  expect(fetchCount()).toBe(1);
});

// nzb-parser.test.ts
it("toLanguageCodes maps indexer names and ffprobe tags", () => {
  expect(toLanguageCodes(["German", "deu", "ger", "English", "und"])).toEqual(["de", "en", "und"]);
  expect(toLanguageCodes(null)).toEqual([]);
});
```

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-api && npx vitest --run src/lib/treasure-maps.test.ts src/lib/nzb-parser.test.ts`
Expected: FAIL (`searchMovie`/`toLanguageCodes` nicht exportiert)

- [ ] **Step 3: Implementieren**

`searchMovie`: wie `searchEpisode`, Cache-Key `tm:movie:${tmdbId}`, Pfad `/movie`, Parameter `tmdbid`, `extended=true`, `limit=100`, **kein** `cat` (so kommen deutsche und internationale Kategorien). German-Muster in `LANGUAGE_PATTERNS` um `deu` erweitern. `withBudgetLock`, `BUDGET_RESERVE`, `toLanguageCodes` verschieben; `episode-requests.ts` importiert sie. `runtime` in `fetchTmdbMovieDetails` aus `detailDe?.runtime || detailEn?.runtime || null`.

- [ ] **Step 4: Ganze API-Suite grün**

Run: `cd openmedia-api && npm test && npm run build`
Expected: alle Tests PASS, `tsc` ohne Fehler

- [ ] **Step 5: Commit**

```bash
git add src/lib/treasure-maps.ts src/lib/nzb-parser.ts src/lib/episode-requests.ts src/lib/tmdb.ts src/lib/*.test.ts
git commit -m "feat(movies): add indexer movie search and share budget/language helpers"
```

---

### Task 2: Gemeinsame Bewertung (`movie-ranking.ts`)

**Files:**
- Create: `openmedia-api/src/lib/movie-ranking.ts`
- Modify: `openmedia-api/src/lib/release-ranking.ts` (nur `export` vor `tierOf`, `bandDistance`, `dropDatalessTwins`, `hasWord`, `hasGermanNameHint`, `includesGerman`, `TIER_RANK`, `Band`; keine Logik-Änderung)
- Test: `openmedia-api/src/lib/movie-ranking.test.ts`

**Interfaces:**
- Consumes: `IndexerRelease` (treasure-maps), `toLanguageCodes` (Task 1)
- Produces:

```ts
export type CandidateGroup = "german" | "foreign" | "cam";
export type CandidateState = "ready" | "available" | "indexer" | "broken";
export const GROUP_RANK: Record<CandidateGroup, number> = { german: 0, foreign: 1, cam: 2 };

export interface OwnFile {
  id: string; hash: string; originalFilename: string; fileSize: bigint | null;
  qualityTier: string | null; videoWidth: number | null;
  audioLanguages: string[]; subtitleLanguages: string[]; releaseType: string | null;
  indexerGuid: string | null; status: string; brokenReason: string | null;
  failedAttempts: number; s3Key: string | null; mediaInfo: unknown;
}

export interface MovieCandidate {
  key: string;                 // "file:<NzbFile.id>" | "guid:<indexer guid>"
  nzbFileId: string | null;
  guid: string | null;
  title: string;
  group: CandidateGroup;
  state: CandidateState;
  qualityTier: QualityTier;
  releaseType: string | null;
  sizeBytes: number | null;
  audioLanguages: string[];    // Codes wie "de", "en"; leer = unbekannt
  germanSubtitles: boolean;
  audioDescription: boolean;
  inTargetBand: boolean;
  hashPrefix: string | null;   // erste 8 Zeichen, nur eigene NZBs
  brokenReason: string | null;
}

export function rankMovieCandidates(
  releases: IndexerRelease[],
  files: OwnFile[],
  opts: { runtimeMinutes: number; band: Band },
): MovieCandidate[];
```

Regeln, die die Tests nicht allein festlegen:
- **Indexer-Treffer:** `password === 2` und `size` nicht `> 0` raus; `dropDatalessTwins`; Treffer mit `guid` gleich einer `OwnFile.indexerGuid` raus (die eigene Datei vertritt ihn). Deutsch = `includesGerman(audioLanguages)` bei Daten, sonst `hasGermanNameHint(title)`. Sprachen-Codes = `toLanguageCodes(audioLanguages)` bei Daten, sonst `parseNzbName(title).audioLanguages`. State `indexer`.
- **Eigene Dateien:** `failedAttempts >= 3` raus. State: `status === "broken"` → `broken` (auch mit `s3Key`), sonst `s3Key` → `ready`, sonst `available`. Deutsch: mit `mediaInfo` → Codes enthalten `de`; ohne → `hasGermanNameHint(originalFilename)`. Sprachen-Codes: mit `mediaInfo` → `toLanguageCodes(audioLanguages)`, ohne → `[]`. Stufe: aus `videoWidth` wie `tierOf`, sonst `qualityTier`, sonst `480p`.
- **Kino-Mitschnitt:** `hasWord(title, ["CAM","HDCAM","TS","HDTS","TELESYNC","TC","TELECINE","SCR","SCREENER","DVDSCR","LD","MD"])` → Gruppe `cam`, egal welche Sprache.
- **Sortierung** (der Reihe nach): `broken` ganz hinten → `GROUP_RANK` → in `cam` zuerst deutsche → Stufe (`inTargetBand ? 5 : TIER_RANK`) → Bonus (Untertitel 2, Hörfilm 1) → State (`ready` < `available` < `indexer`) → wie `rankReleases` (im Band größer zuerst, sonst kleinerer Band-Abstand). Unbekannte Größe gilt als außerhalb des Bands mit Abstand `Infinity`.

- [ ] **Step 1: Failing Tabellentests schreiben** (`release()`-Helfer wie in `release-ranking.test.ts`, `file()`-Helfer für `OwnFile`; `opts = { runtimeMinutes: 120, band: { min: 1.5, max: 6 } }`)

```ts
it("German 720p beats English 1080p", ...)            // keys: ["guid:de720", "guid:en1080"]
it("cam goes after foreign; German cam before English cam", ...)
  // Eingabe: German.TS.1080p, English.1080p.WEB, English.CAM → ["guid:en-web","guid:de-ts","guid:en-cam"]
it("tie in group, tier and bonus: ready file beats bigger indexer hit", ...)
  // file 1080p 6 GB s3Key gesetzt vs. Treffer 1080p 10 GB → file zuerst
it("bigger tier still beats a ready file", ...)        // ready 720p vs Treffer 1080p im Band → Treffer zuerst
it("drops the indexer hit that an own file already represents", ...)
it("drops password-protected releases and files with 3 failures", ...)
it("broken own file is listed last with state broken even when on S3", ...)
it("own file without language hint and without mediaInfo is foreign with no languages", () => {
  const [c] = rankMovieCandidates([], [file({ originalFilename: "Film.2024.1080p.WEB.mkv", mediaInfo: null })], opts);
  expect(c.group).toBe("foreign");
  expect(c.audioLanguages).toEqual([]);
});
it("uses ffprobe languages for own files with mediaInfo", ...) // audioLanguages ["ger","eng"] → group german, codes ["de","en"]
it("DTS in a title is not a cam release", ...)           // "Film.German.DTS.1080p.BluRay" → group german
```

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-ranking.test.ts`
Expected: FAIL (Modul fehlt)

- [ ] **Step 3: `rankMovieCandidates` nach den Regeln oben implementieren**

- [ ] **Step 4: Tests grün, Serien-Tests unverändert grün**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-ranking.test.ts src/lib/release-ranking.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/movie-ranking.ts src/lib/movie-ranking.test.ts src/lib/release-ranking.ts
git commit -m "feat(movies): rank own NZBs and indexer hits together"
```

---

### Task 3: Film anfordern (`MovieRequest` + `requestMovie`)

**Files:**
- Modify: `openmedia-api/prisma/schema.prisma` (neues Modell, Relationen an `User` und `NzbFile`)
- Create: Migration über `npx prisma migrate dev --name add_movie_requests`
- Create: `openmedia-api/src/lib/download-jobs.ts`
- Modify: `openmedia-api/src/routes/downloads.ts:247-345` (`POST /jobs` nutzt `startDownloadJob`)
- Create: `openmedia-api/src/lib/movie-requests.ts`
- Test: `openmedia-api/src/lib/movie-requests.test.ts`

**Interfaces:**
- Consumes: Task 1 (`searchMovie`, `withBudgetLock`, `BUDGET_RESERVE`, `toLanguageCodes`, `runtime`), Task 2 (`rankMovieCandidates`, `MovieCandidate`, `GROUP_RANK`), `ingestNzb`, `downloadNzb`, `getDownloadBudget`, `searchTmdbMovieById`, `getTargetBand`
- Produces:

```prisma
model MovieRequest {
  id           String   @id @default(uuid())
  tmdbId       Int      @map("tmdb_id")
  status       String   // 'downloading' | 'done' | 'failed'
  allowedGroup String   @map("allowed_group")  // 'german' | 'foreign' | 'cam'
  manual       Boolean  @default(false)
  attempts     Int      @default(0)
  triedKeys    String[] @default([]) @map("tried_keys")
  failReason   String?  @map("fail_reason")    // 'budget' | 'no_candidate' | 'download_failed'
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId       String   @map("user_id")
  nzbFile      NzbFile? @relation(fields: [nzbFileId], references: [id], onDelete: SetNull)
  nzbFileId    String?  @map("nzb_file_id")
  @@unique([userId, tmdbId])
  @@index([status])
  @@index([nzbFileId])
  @@map("movie_requests")
}
```

```ts
// download-jobs.ts — aus POST /downloads/jobs herausgelöst, Verhalten der Route unverändert
export function startDownloadJob(nzbFileId: string, userId: string | null):
  Promise<{ kind: "created"; job: DownloadJob & { nzbFile: NzbFile & { movie: NzbMovie | null } } }
        | { kind: "conflict"; existingJob: DownloadJob }>;
// enthält Serializable-Transaktion und das Auto-Provisioning (AUTO_PROVISION !== "false")

// movie-requests.ts
export const MAX_ATTEMPTS = 3;
export const DEFAULT_MOVIE_RUNTIME_MINUTES = 110;
export function listMovieCandidates(tmdbId: number):
  Promise<{ candidates: MovieCandidate[]; indexer: "ok" | "unavailable" }>;
export type RequestOutcome =
  | { status: "ready"; nzbFileId: string }
  | { status: "downloading"; nzbFileId: string }
  | { status: "confirm_required"; group: "foreign" | "cam" }
  | { status: "budget" }
  | { status: "candidate_gone" }
  | { status: "failed" };
export function requestMovie(userId: string, tmdbId: number, key: string,
  opts: { confirmed: boolean; manual: boolean }): Promise<RequestOutcome>;
```

Ablauf `listMovieCandidates`: eigene Dateien über `nzbMovie.findUnique({ where: { tmdbId }, include: { nzbFiles } })`; Laufzeit aus `searchTmdbMovieById` (`found` → `runtime`, sonst 110); `searchMovie` — `IndexerError` → `[]` und `indexer: "unavailable"`; Indexer-Treffer, deren `guid` als `broken` `NzbFile` existiert, vorher raus (wie `rankEpisodeCandidates`).

Ablauf `requestMovie`:
1. Kandidat per `key` suchen (nicht `broken`), sonst `candidate_gone`.
2. Gruppe ≠ `german` und `!confirmed` → `confirm_required` (nichts angelegt).
3. `MovieRequest` upserten (`userId_tmdbId`): `status "downloading"`, `allowedGroup = group`, `manual`, `attempts 0`, `triedKeys []`, `failReason null`, `nzbFileId null`.
4. `attemptCandidates(request, [gewählter, ...Rest])` — interne Funktion, auch für Task 4. Rest = nur wenn `!manual`: nicht `broken`, nicht in `triedKeys`, `GROUP_RANK[c.group] <= GROUP_RANK[allowedGroup]`. Pro Kandidat `attempts++`, `key` in `triedKeys`; Schluss bei `attempts >= MAX_ATTEMPTS`.
   - `ready` → Request `done` + `nzbFileId`, `UserLibrary` upsert (wie `addToLibrary` in `episode-requests.ts`) → `ready`.
   - `available` → `startDownloadJob(nzbFileId, userId)`; `created`/`conflict` → `downloading`.
   - `indexer` → in `withBudgetLock`: `getDownloadBudget()`, bei `used >= max - BUDGET_RESERVE` → Request `failed`/`budget` → `budget`; sonst `downloadNzb` + `ingestNzb` mit `resolve: async () => ({ source: "tmdb-found", tmdb })` und `fileData` wie `ingestCandidate` in `episode-requests.ts`. Ergebnis-Zuordnung wie dort (`created`/`conflict`/`hash-race` → `downloading`, `already-available` → `ready`, `broken` → nächster Kandidat, sonst `failed`).
   - Keiner übrig → Request `failed`/`no_candidate` → `failed`.
   TMDB für den Ingest nicht `found` → `TmdbUnavailableError` aus `episode-requests.ts` werfen.

- [ ] **Step 1: Failing tests schreiben** (Mocks wie in `episode-requests.test.ts`: `tmdb.js`, `treasure-maps.js` mit `searchMovie`, `nzb-service.js`; `ttl-cache` per `clearCache()` in `beforeEach`)

```ts
it("one click on a German indexer hit downloads the NZB and starts a job", ...)
  // expect outcome {status:"downloading"}, downloadNzb called once, MovieRequest.status "downloading", DownloadJob exists
it("asks before loading an English-only version", async () => {
  expect(await requestMovie(u.id, 603, "guid:en1080", { confirmed: false, manual: false }))
    .toEqual({ status: "confirm_required", group: "foreign" });
  expect(mockDownload).not.toHaveBeenCalled();
});
it("ready own file returns ready and adds it to the library", ...)
it("own NZB without S3 starts a job without touching the indexer budget", ...) // getDownloadBudget not called
it("stops at the budget reserve", ...)              // budget {used:350,max:400} → {status:"budget"}, failReason "budget"
it("unknown key returns candidate_gone", ...)
it("broken ingest tries the next German candidate but never an English one", ...)
  // Kandidaten: de1 (ingest broken), en1 → outcome "failed", downloadNzb nur für de1
it("manual pick does not fall back to another version", ...)
it("second user clicking the same film shares the active job", ...)
  // beide Requests downloading mit gleicher nzbFileId
```

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-requests.test.ts`
Expected: FAIL (Modul fehlt)

- [ ] **Step 3: Schema + Migration, `startDownloadJob`, `movie-requests.ts` implementieren**

- [ ] **Step 4: Tests grün, `downloads.test.ts` unverändert grün**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-requests.test.ts src/routes/downloads.test.ts && npm run build`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add prisma src/lib/download-jobs.ts src/lib/movie-requests.ts src/lib/movie-requests.test.ts src/routes/downloads.ts
git commit -m "feat(movies): request a film from own NZBs or the indexer"
```

---

### Task 4: Weiterprobieren im Hintergrund (`syncMovieRequests`)

**Files:**
- Modify: `openmedia-api/src/lib/movie-requests.ts`
- Modify: `openmedia-api/src/lib/job-reconciler.ts:~550` (Aufruf direkt nach `syncEpisodeRequests`, eigenes try/catch wie dort)
- Test: `openmedia-api/src/lib/movie-requests.test.ts`

**Interfaces:**
- Consumes: `attemptCandidates` (Task 3), `listMovieCandidates`
- Produces: `syncMovieRequests(): Promise<void>`

Ablauf:
1. **`downloading`:** `nzbFile.s3Key` gesetzt → `done` + `UserLibrary` für `request.userId`. Keine aktive Job (`queued|provisioning|downloading|uploading|needs_review`) mehr → `manual` → `failed`/`download_failed`; sonst `attemptCandidates` mit dem Rest der frischen Liste.
2. **`done`, `allowedGroup "german"`, `nzbFile.mediaInfo` gesetzt:** Codes = `toLanguageCodes(nzbFile.audioLanguages)` ohne `und`. Sind sie nicht leer und enthalten kein `de` → `NzbFile` `status "broken"`, `brokenReason "Kein deutscher Ton (ffprobe)"`; `UserLibrary` dieses Nutzers für diese Datei `removedAt = now`; dann `manual` → `failed`/`download_failed`, sonst `status "downloading"` und `attemptCandidates` mit dem Rest.
   Abfrage so eingrenzen, dass geprüfte Anforderungen nicht jedes Mal neu geladen werden: Prisma-Filter `nzbFile: { mediaInfo: { not: Prisma.DbNull }, NOT: { audioLanguages: { hasSome: ["de", "ger", "deu"] } } }`.
3. Höchstens 20 Anforderungen pro Lauf (wie `SYNC_BATCH`).

- [ ] **Step 1: Failing tests schreiben**

```ts
it("adds the film to every requester's library when the shared job completes", ...)  // Review Focus 3
it("retries the next German candidate after the job failed", ...)
it("marks a ffprobe-English 'German' file broken, removes it from the library and retries", ...)
it("does not mark broken when ffprobe has no or only 'und' language tags", async () => {
  // nzbFile.mediaInfo = {}, audioLanguages = ["und"] → status bleibt "done", NzbFile.status nicht "broken"
});
it("does not retry a manual pick after failure", ...)  // status "failed", failReason "download_failed"
it("gives up after 3 attempts", ...)
```

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-requests.test.ts`
Expected: FAIL (`syncMovieRequests` fehlt)

- [ ] **Step 3: `syncMovieRequests` implementieren und im Reconciler aufrufen**

- [ ] **Step 4: Tests grün inkl. Reconciler**

Run: `cd openmedia-api && npx vitest --run src/lib/movie-requests.test.ts src/lib/job-reconciler.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/movie-requests.ts src/lib/movie-requests.test.ts src/lib/job-reconciler.ts
git commit -m "feat(movies): retry failed film downloads and re-check German audio"
```

---

### Task 5: API-Routen `/movies`

**Files:**
- Create: `openmedia-api/src/routes/movies.ts`
- Create: `openmedia-api/src/routes/indexer-errors.ts` (verschiebt `handleError` aus `series.ts`, neuer Parameter `fallbackMessage: string`)
- Modify: `openmedia-api/src/routes/series.ts` (nutzt `handleIndexerError(err, res, context, "Interner Fehler bei der Serien-Anfrage.")`)
- Modify: `openmedia-api/src/app.ts:35` (`app.use("/movies", movieRoutes)`)
- Test: `openmedia-api/src/routes/movies.test.ts`

**Interfaces:**
- Consumes: `listMovieCandidates`, `requestMovie`, `RequestOutcome` (Task 3)
- Produces (JSON, alle Routen hinter `requireAuth`):
  - `GET /movies/candidates?tmdbId=603` → `200 { candidates: MovieCandidate[], indexer: "ok" | "unavailable" }`
  - `POST /movies/requests` Body `{ tmdbId: number, key: string, confirmed?: boolean, manual?: boolean }`
    - `ready`/`downloading` → `200 { status, nzbFileId }`
    - `confirm_required` → `409 { reason: "confirm_required", group }`
    - `budget` → `429 { reason: "budget", error: "Tageslimit erreicht — morgen wieder" }`
    - `candidate_gone` → `404 { reason: "candidate_gone", error: "Version nicht mehr verfügbar" }`
    - `failed` → `409 { reason: "failed", error: "Keine passende Version konnte geladen werden" }`
  - Validierung: `tmdbId` positive Ganzzahl, `key` passt auf `/^(file:[0-9a-f-]{36}|guid:[a-f0-9]{32})$/`, sonst `400`.

- [ ] **Step 1: Failing tests schreiben** (supertest + `createApp`, `movie-requests.js` gemockt)

```ts
it("GET /movies/candidates returns own files when the indexer is not configured", ...)  // Review Focus 4: 200, indexer "unavailable"
it("GET /movies/candidates rejects a bad tmdbId with 400", ...)
it("POST /movies/requests maps confirm_required to 409", ...)
it("POST /movies/requests maps budget to 429 with the German message", ...)
it("POST /movies/requests maps candidate_gone to 404", ...)                              // Review Focus 5
it("POST /movies/requests rejects a malformed key with 400", ...)
it("requires login", ...)                                                                // 401 ohne Token
```

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-api && npx vitest --run src/routes/movies.test.ts`
Expected: FAIL (404 für `/movies/...`)

- [ ] **Step 3: Routen, Fehler-Helfer und Mount implementieren**

- [ ] **Step 4: Ganze API-Suite + Build grün**

Run: `cd openmedia-api && npm test && npm run build`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/routes/movies.ts src/routes/movies.test.ts src/routes/indexer-errors.ts src/routes/series.ts src/app.ts
git commit -m "feat(movies): expose film candidates and requests"
```

---

### Task 6: Film-Knopf in der Webseite

**Files:**
- Modify: `openmedia-web/src/lib/backend.ts` (Typ + zwei Aufrufe)
- Modify: `openmedia-web/src/contexts/download-context.tsx` (zwei Funktionen im Context-Wert)
- Modify: `openmedia-web/src/components/movie/download-button.tsx` (Datenquelle, Zustände, Liste, Rückfrage)
- Test: `openmedia-web/src/components/movie/download-button.test.tsx`, `openmedia-web/src/components/preview/movie-preview-actions.test.tsx`

**Interfaces:**
- Consumes: Routen aus Task 5
- Produces:

```ts
// backend.ts — MovieCandidate wie API-JSON (Task 2), plus:
export function getMovieCandidates(tmdbId: number, token: string):
  Promise<ApiResult<{ candidates: MovieCandidate[]; indexer: "ok" | "unavailable" }>>;
export function requestMovieDownload(
  body: { tmdbId: number; key: string; confirmed?: boolean; manual?: boolean }, token: string,
): Promise<ApiResult<{ status: "ready" | "downloading"; nzbFileId: string; reason?: string; group?: string; error?: string }>>;

// download-context.tsx — im Context-Wert
loadMovieCandidates: (tmdbId: number) => Promise<MovieCandidate[] | null>;
requestMovie: (body: { tmdbId: number; key: string; confirmed?: boolean; manual?: boolean }) =>
  Promise<{ ok: boolean; status: number; reason?: string }>;   // ruft nach Erfolg `refresh()`
```

(`ApiResult` = der Rückgabetyp, den `backendFetch` schon hat.)

Festgelegte Texte und Regeln:
- Sprachnamen: `{ de: "Deutsch", en: "Englisch", fr: "Französisch", es: "Spanisch", it: "Italienisch", ja: "Japanisch", ko: "Koreanisch", ru: "Russisch", pt: "Portugiesisch", nl: "Niederländisch" }`.
- Schild: `cam` → „Kino-Mitschnitt"; `foreign` mit genau einer bekannten Sprache ≠ `de` → „Nur {Name}", sonst „Kein Deutsch"; `german` → keins. Gelbes Schild (`bg-amber-500/25 text-amber-400`).
- Haupt-Knopf (bester nicht-`broken` Kandidat = `candidates[0]`): Ladezustand „Suche Versionen…"; leer → „Keine Version gefunden" (mit `hideWhenUnavailable` → `null`); `ready` → gold „Herunterladen" + Stufe; aktiver Job zur `nzbFileId` → Status-Label + `%` wie heute; sonst grün „Verfügbar machen" + Stufe + Schild, Icon `CloudDownload` (`size-4`). Nach `429` zeigt der Knopf deaktiviert „Tageslimit erreicht — morgen wieder".
- Pfeil (`ChevronDown`) neben dem Haupt-Knopf öffnet immer die Liste; Gruppen in dieser Reihenfolge, leere weglassen: „Deutsch", „Kein Deutsch", „Kino-Mitschnitt", „Fehlerhaft" (ausgegraut, nicht klickbar, mit `brokenReason`).
- Zeile Titel: Indexer `"{Stufe} · {releaseType ?? "Unbekannt"} · {GB, de-DE, 1 Nachkommastelle} GB"`; eigene NZB `"{Stufe} · eigene NZB · {hashPrefix}"`. Unterzeile mit `" · "` verbunden: Sprachen mit `" + "` verbunden (leer → „Sprache unbekannt"), „Untertitel DE", „Hörfilm", „schon im Bestand" (eigene NZB), „Empfohlen" (nur `candidates[0]`). Rechts: „Download" (`ready`), „Abbrechen" (aktiver Job), sonst „Starten".
- Klick Haupt-Knopf → `requestMovie({ tmdbId, key })`; Klick Zeile → zusätzlich `manual: true`. Gruppe ≠ `german` → erst `AlertDialog` aus `components/ui/alert-dialog.tsx`, dann mit `confirmed: true`:
  - `foreign`, eine Sprache: Titel „Kein deutscher Ton", Text „Diesen Film gibt es bisher nur auf {Name}. Trotzdem laden?", Knöpfe „Abbrechen" / „{Name} laden".
  - `foreign`, sonst: gleicher Titel, Text „Diesen Film gibt es bisher nicht auf Deutsch. Trotzdem laden?", Knopf „Trotzdem laden".
  - `cam`: Titel „Kino-Mitschnitt", Text „Diese Version wurde im Kino abgefilmt. Bild und Ton sind schlecht. Trotzdem laden?", Knopf „Trotzdem laden".
- `404 candidate_gone` → Toast „Version nicht mehr verfügbar" und Liste neu laden. Liste auch neu laden, wenn ein Job zu einer Kandidaten-`nzbFileId` `completed` oder `failed` wird (das bestehende `completedJobIds`-Muster um `failed` erweitern).

- [ ] **Step 1: Failing tests schreiben** (Context-Mock wie heute, `loadMovieCandidates`/`requestMovie` statt `checkAvailability`)

```tsx
it("shows 'Keine Version gefunden' when nothing is found", ...)
it("renders nothing with hideWhenUnavailable when nothing is found", ...)
it("one click on a German candidate requests it without asking", async () => {
  // candidates [de1080 indexer] → Klick „Verfügbar machen" → requestMovie({ tmdbId: 603, key: "guid:de1080" })
});
it("marks an English-only version and asks before loading", async () => {
  // Schild „Nur Englisch" sichtbar; Klick → Dialog „Kein deutscher Ton"; „Englisch laden" → requestMovie mit confirmed: true
});
it("marks a cinema rip and asks before loading", ...)
it("dropdown groups German, non-German and cam rows with quality lines", ...)
  // „1080p · WEB-DL · 7,8 GB", „Deutsch + Englisch · Untertitel DE · Empfohlen", „1080p · eigene NZB · a3f9c21e"
it("a row click sends manual: true", ...)
it("shows the daily limit after a 429", ...)
it("reloads the list and toasts after candidate_gone", ...)   // Review Focus 5
it("shows gold 'Herunterladen' for a ready best candidate", ...)
```

Und in `movie-preview-actions.test.tsx` die Mocks auf `loadMovieCandidates` umstellen; Erwartung „kein Knopf, wenn nichts gefunden" bleibt.

- [ ] **Step 2: Tests laufen lassen, sie schlagen fehl**

Run: `cd openmedia-web && npx vitest --run src/components/movie/download-button.test.tsx src/components/preview/movie-preview-actions.test.tsx`
Expected: FAIL

- [ ] **Step 3: `backend.ts`, Context und `download-button.tsx` umsetzen**

`checkAvailability` bleibt im Context, falls andere Stellen es nutzen (`grep -rn checkAvailability src` prüfen; ungenutzt → entfernen).

- [ ] **Step 4: Web-Suite, Lint und Build grün**

Run: `cd openmedia-web && npm test && npm run lint && npm run build`
Expected: PASS, keine Lint-Fehler, Build ok

- [ ] **Step 5: Commit**

```bash
git add src/lib/backend.ts src/contexts/download-context.tsx src/components/movie/download-button.tsx src/components/movie/download-button.test.tsx src/components/preview/movie-preview-actions.test.tsx
git commit -m "feat(movies): one-click film download with indexer fallback and version picker"
```

---

### Task 7: Echter Durchlauf

**Files:** keine neuen; nur Prüfung.

- [ ] **Step 1: Echte Indexer-Antwort prüfen**

Mit lokalem API und eingetragenem `treasureMapsApiKey`: `GET /movies/candidates?tmdbId=603` (Matrix) und für einen aktuellen Kinofilm. Erwartet: deutsche Treffer oben, `cam` erkannt, `indexer: "ok"`. Kommen kaum deutsche Treffer, dann prüfen, ob `/movie` ohne `cat` die Kategorie `2100` mitliefert; sonst in `searchMovie` zwei Abfragen (`cat=2000` und `cat=2100`) zusammenführen und Test anpassen.

- [ ] **Step 2: Im Browser prüfen** (`openmedia-web` lokal, angemeldet)

Film-Seite und Vorschau-Kachel: Zustände aus Spec 6.7 sichtbar, Rückfrage erscheint bei „Nur Englisch", Liste zeigt eigene NZB und Treffer. Screenshot der Liste und des Dialogs anhängen.

- [ ] **Step 3: PRs öffnen** (je Repo einer, Beschreibung verweist auf Spec und Plan)

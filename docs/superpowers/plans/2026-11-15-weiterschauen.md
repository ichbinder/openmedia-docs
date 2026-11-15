# Weiterschauen Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** openmedia merkt sich pro Konto und Titel die Stelle, fragt beim Start „Weiterschauen ab mm:ss?“, zeigt eine Zeile „Weiterschauen“ auf der Startseite und spielt am Folgen-Ende nach 10 s Countdown die nächste Folge — im Web und auf tvOS.

**Architecture:** Das API speichert den Fortschritt in der neuen Tabelle `watch_progress` (pro Titel, nicht pro Datei). Reine Regeln (Zustand, Übertragung auf andere Länge, beste Datei) liegen in `src/lib/watch-progress.ts`, Datenbank- und TMDB-Logik in `src/lib/progress-service.ts`, die Route in `src/routes/progress.ts`. Beide Player melden über `PUT /progress`, lesen `resume` aus `stream-link` und holen die nächste Folge über `GET /progress/next/:nzbFileId`. Der Web-Player ist danach selbstständig: wer ihn mit einer `nzbFileId` öffnet, bekommt Fortsetzen, Melden und Folgen-Kette ohne weitere Arbeit.

**Tech Stack:** Express 5, Prisma 7, PostgreSQL, Vitest + supertest (echte Test-DB über `src/test/setup.ts`); Next.js 16, React 19, Vidstack, Vitest + Testing Library; SwiftUI/AVKit tvOS 18, Swift Testing in `OpenMediaKit` (`swift test`).

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-15-weiterschauen-design.md`

**Repos/Branches:** `openmedia-api`, `openmedia-web`, `openmedia-tvos` (alle unter `/Users/jschattka/git_private/cinescope/`), jeweils Branch `feature/weiterschauen` von frischem `main`.

## Global Constraints

- Alle Texte für Nutzer auf Deutsch, mit echten Umlauten.
- Keine neuen Abhängigkeiten in allen drei Repos.
- Zahlen aus Spec 6.1, pro Repo an **einer** Stelle als Konstanten: Melden alle `30` s; „fertig“ ab `0.9` der Länge; Frage „Fortsetzen?“ nur bei Stelle `> 60` s und nicht fertig; Knopf „Nächste Folge ▶“ in den letzten `60` s; Countdown `10` s; Längen-Toleranz `0.02`; beim Fortsetzen `5` s zurück; Zeile max. `20` Einträge.
- Zustand: **angefangen** = Stelle > 60 s und < 90 %; sonst **fertig**, wenn `finishedAt` gesetzt; sonst **neu**.
- Fortschritt hängt am Titel (`movieId` oder `episodeId`), nie nur an der Datei. `nzbFileId` ist `onDelete: SetNull`.
- Kein Profil-/Personen-Feld. Keine Wertung, keine „abgebrochen“-Logik.
- Migration nur additiv; bestehende Tests in allen Repos bleiben grün.
- `/progress`-Antworten immer mit JSON-Body (kein 204), weil der Web-Proxy jede Antwort als JSON liest.
- Auslieferung: API (mit Migration) → Web → tvOS.

## Review Focus

1. Browser-Tab wird mitten im Film geschlossen → die letzte Stelle kommt per `sendBeacon` an, höchstens ~30 s fehlen (Test in Task 4).
2. Schnelles Spulen mit der Leiste → nicht eine Meldung pro Sprung, sondern eine nach Ruhe (Test in Task 4).
3. Nächste Folge liegt in der nächsten Staffel, ist noch nicht ausgestrahlt, oder TMDB ist weg → richtige Folge bzw. kein Eintrag; `GET /progress/continue` antwortet trotzdem 200 (Tests in Task 3).
4. Andere Version mit anderer Länge (Kinofassung vs. Extended) → Stelle wird prozentual übertragen, 5 s zurück, nie negativ (Test in Task 1).
5. Player meldet beim Start Länge `0`/`NaN` oder eine Stelle hinter dem Ende → API antwortet 400, Clients melden ohne bekannte Länge gar nicht (Tests in Task 2, 4 und 7).

---

### Task 1: Datenmodell und reine Regeln (API)

**Files:**
- Modify: `openmedia-api/prisma/schema.prisma` (neues Modell, Rückrelationen an `User`, `NzbMovie`, `NzbEpisode`, `NzbFile`)
- Create: `openmedia-api/prisma/migrations/<timestamp>_add_watch_progress/migration.sql` (per `npx prisma migrate dev --name add_watch_progress`)
- Modify: `openmedia-api/src/test/setup.ts` (Tabelle `watch_progress` in die TRUNCATE-Liste)
- Create: `openmedia-api/src/lib/watch-progress.ts`
- Modify: `openmedia-api/src/lib/episode-requests.ts` (`tierRank` exportieren)
- Test: `openmedia-api/src/lib/watch-progress.test.ts`

**Interfaces:**
- Produces:
  - Prisma-Modell `WatchProgress` (`@@map("watch_progress")`): `id` uuid, `userId`, `movieId String?`, `episodeId String?`, `nzbFileId String?`, `positionSec Int`, `durationSec Int`, `finishedAt DateTime?`, `startedAt DateTime @default(now())`, `updatedAt DateTime @updatedAt`, `hiddenAt DateTime?`; Spaltennamen snake_case per `@map` wie im Rest des Schemas; `@@unique([userId, movieId])`, `@@unique([userId, episodeId])`, `@@index([userId, updatedAt])`; `user`, `movie`, `episode` mit `onDelete: Cascade`, `nzbFile` mit `onDelete: SetNull`.
  - `export const PROGRESS = { REPORT_INTERVAL_SEC: 30, FINISHED_RATIO: 0.9, RESUME_MIN_SEC: 60, NEXT_BUTTON_SEC: 60, COUNTDOWN_SEC: 10, LENGTH_TOLERANCE: 0.02, RESUME_REWIND_SEC: 5, CONTINUE_LIMIT: 20 } as const`
  - `export type TitleState = "new" | "started" | "finished"`
  - `export function isFinishedPosition(positionSec: number, durationSec: number): boolean`
  - `export function titleState(p: { positionSec: number; durationSec: number; finishedAt: Date | null }): TitleState`
  - `export function resumePosition(saved: { positionSec: number; durationSec: number }, fileDurationSec: number | null): number` — ganze Sekunden
  - `export function pickStreamableFile<T extends { s3StreamKey: string | null; status: string; qualityTier: string | null; resolution: string | null; fileSize: bigint | null }>(files: T[]): T | undefined` — gleiche Regel wie Web `bestVersion(files, "s3StreamKey")` und tvOS `bestStreamFile`: nur mit `s3StreamKey` und `status !== "broken"`, beste Qualität (`tierRank(qualityTier ?? resolution)`), dann größte Datei.

- [ ] **Step 1: Write the failing tests** in `watch-progress.test.ts`:
  - `isFinishedPosition(5400, 6000)` → `true`; `isFinishedPosition(5399, 6000)` → `false`.
  - `titleState({positionSec: 61, durationSec: 6000, finishedAt: null})` → `"started"`; `positionSec: 60` → `"new"`; `positionSec: 5500, finishedAt: null` → `"new"` (fertig zählt nur über `finishedAt`); `positionSec: 5500, finishedAt: new Date()` → `"finished"`; `positionSec: 30, finishedAt: new Date()` → `"finished"`; `positionSec: 120, finishedAt: new Date()` → `"started"` (Wiederholung).
  - `resumePosition({positionSec: 3000, durationSec: 6000}, 6060)` → `2995` (1 % Unterschied, 1:1 minus 5); `…, 7200)` → `3595` (prozentual: 3600 − 5); `…, null)` → `2995`; `resumePosition({positionSec: 3, durationSec: 6000}, 6000)` → `0`.
  - `pickStreamableFile`: ignoriert Datei ohne `s3StreamKey` und `status: "broken"`; bevorzugt `2160p` vor `1080p`; bei gleicher Qualität die größere `fileSize`; leere Liste → `undefined`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/watch-progress.test.ts` — Expected: FAIL (Modul fehlt).
- [ ] **Step 3: Implement** Modell + Migration + `setup.ts`-Eintrag + `watch-progress.ts`; `tierRank` aus `episode-requests.ts` exportieren und benutzen.
- [ ] **Step 4: Run** `npx vitest --run src/lib/watch-progress.test.ts` — PASS; dann `npm test` — alles grün.
- [ ] **Step 5: Commit** `feat(progress): add watch_progress table and pure progress rules`

### Task 2: Fortschritt melden und `resume` im Stream-Link (API)

**Files:**
- Create: `openmedia-api/src/lib/progress-service.ts`
- Create: `openmedia-api/src/routes/progress.ts`
- Modify: `openmedia-api/src/app.ts` (`app.use("/progress", progressRoutes)`)
- Modify: `openmedia-api/src/routes/nzb.ts` (`GET /files/:id/stream-link` → Feld `resume`)
- Test: `openmedia-api/src/routes/progress.test.ts`, Ergänzung in `openmedia-api/src/routes/nzb.test.ts`

**Interfaces:**
- Consumes: `PROGRESS`, `isFinishedPosition`, `titleState`, `resumePosition` (Task 1).
- Produces:
  - `export type Resume = { positionSec: number; durationSec: number; finished: boolean }`
  - `export type RecordResult = { status: "ok"; progress: Resume } | { status: "invalid" } | { status: "not_found" }`
  - `export async function recordProgress(userId: string, nzbFileId: string, positionSec: number, durationSec: number): Promise<RecordResult>` — bestimmt `movieId`/`episodeId` aus der Datei (Datei ohne beides → `not_found`), upsert, setzt `finishedAt` beim ersten Erreichen von 90 % (bleibt danach), setzt `hiddenAt = null`, speichert `nzbFileId`.
  - `export async function getResume(userId: string, file: { movieId: string | null; episodeId: string | null; duration: number | null }): Promise<Resume | null>` — `null`, wenn kein Eintrag oder Zustand nicht `"started"`; sonst `positionSec` über `resumePosition` auf diese Datei umgerechnet.
  - Route `PUT /progress` und `POST /progress`, Body `{ nzbFileId: string, positionSec: number, durationSec: number }` → 200 `{ progress: Resume }`; 400 `{ error }` bei nicht-endlichen Zahlen, `positionSec < 0`, `durationSec <= 0`, `positionSec > durationSec * (1 + LENGTH_TOLERANCE) + 5`; 404 bei unbekannter Datei. Werte werden auf ganze Sekunden gerundet.
  - `stream-link`-Antwort zusätzlich `resume: Resume | null`.

- [ ] **Step 1: Write the failing tests** (Muster wie `watchlist.test.ts`; `NzbMovie`/`NzbShow`/`NzbEpisode`/`NzbFile` direkt per `prisma` anlegen):
  - `PUT` mit Film-Datei legt Eintrag an; zweiter `PUT` aktualisiert denselben Eintrag (genau 1 Zeile für User+Film).
  - Zwei Dateien desselben Films → weiter 1 Zeile, `nzbFileId` = zuletzt gemeldete.
  - `POST /progress` mit `Content-Type: application/json` wirkt wie `PUT`.
  - `positionSec: 5400, durationSec: 6000` → `progress.finished === true`; danach `positionSec: 30` → `finished === true` bleibt in der DB (`finishedAt` nicht `null`).
  - 400 für `durationSec: 0`, `positionSec: -1`, `positionSec: "abc"`, `positionSec: 7000, durationSec: 6000`; 404 für unbekannte `nzbFileId`; 401 ohne Token.
  - Konto B sieht Konto As Stand nicht: `stream-link` für B liefert `resume: null`.
  - `stream-link`: nach `PUT 3000/6000` liefert dieselbe Datei `resume.positionSec === 2995`; eine zweite Datei mit `duration: 7200` liefert `3595`; nach `PUT 5500/6000` (fertig) und neu `PUT 20/6000` → `resume: null`.
- [ ] **Step 2: Run** `npx vitest --run src/routes/progress.test.ts src/routes/nzb.test.ts` — FAIL.
- [ ] **Step 3: Implement** `progress-service.ts`, `routes/progress.ts`, Einbindung in `app.ts`, `resume` in `nzb.ts`.
- [ ] **Step 4: Run** dieselben Tests — PASS; `npm test` — grün.
- [ ] **Step 5: Commit** `feat(progress): record watch progress and return resume with stream link`

### Task 3: Nächste Folge, Zeile „Weiterschauen“, Entfernen, Fortschritt auf Detailseiten (API)

**Files:**
- Modify: `openmedia-api/src/lib/progress-service.ts`
- Modify: `openmedia-api/src/routes/progress.ts`
- Modify: `openmedia-api/src/routes/nzb.ts` (`/movies/by-tmdb` und `/shows/by-tmdb` → Feld `myProgress`)
- Test: `openmedia-api/src/routes/progress.test.ts` (TMDB per `vi.mock("../lib/tmdb.js")`, nur `getTmdbSeasonById` gemockt)

**Interfaces:**
- Consumes: Task 1 + 2; `getTmdbSeasonById(tmdbId, season): Promise<TmdbSeasonByIdResult>` aus `src/lib/tmdb.ts`.
- Produces:
  - `export type EpisodeAvailability = { nzbFileId: string | null; request: { status: string; waitReason: string | null } | null }`
  - `export type NextEpisode = { showTmdbId: number; season: number; episode: number; titleDe: string } & EpisodeAvailability`
  - `export async function findNextEpisode(userId: string, showTmdbId: number, season: number, episode: number): Promise<NextEpisode | null>` — nächste Nummer in derselben Staffel, sonst Folge 1 der nächsten Staffel; nur wenn `airDate` ≤ heute (UTC-Datum). TMDB-Status `error`/`disabled` oder `not_found` → `null`, nie werfen. Verfügbarkeit: `NzbEpisode` per (`showId`, `season`, `episode`) — fehlt die Zeile, `nzbFileId: null, request: null`; sonst `pickStreamableFile` über deren Dateien und eine offene `EpisodeRequest` des Kontos.
  - `export type ContinueItem = { kind: "movie" | "episode"; tmdbId: number; title: string; posterPath: string | null; season: number | null; episode: number | null; episodeTitle: string | null; positionSec: number; durationSec: number; updatedAt: string } & EpisodeAvailability` — `tmdbId` ist bei Folgen die Serien-ID; für eine noch nicht begonnene nächste Folge `positionSec: 0, durationSec: 0`; `request` bei Filmen aus `MovieRequest` (`status`, `waitReason: failReason`).
  - `export async function listContinue(userId: string): Promise<ContinueItem[]>` — Regeln aus Spec 6.3: Filme im Zustand `started` und nicht versteckt; pro Serie nur die Folgen-Zeile mit neuestem `updatedAt`; ist sie versteckt → nichts; `started` → diese Folge; `finished` → `findNextEpisode`, `null` → nichts. Sortiert nach `updatedAt` absteigend, max. `PROGRESS.CONTINUE_LIMIT`.
  - `export async function hideContinue(userId: string, kind: "movie" | "show", tmdbId: number): Promise<number>` — setzt `hiddenAt` (Film bzw. alle Folgen-Zeilen der Serie), liefert Anzahl.
  - Routen: `GET /progress/continue` → `{ items: ContinueItem[] }`; `GET /progress/next/:nzbFileId` → `{ next: NextEpisode | null }` (Film-Datei → `null`, unbekannte Datei → 404); `DELETE /progress/:kind/:tmdbId` (`kind` ∈ `movie`, `show`, sonst 400) → `{ hidden: number }`.
  - `/nzb/movies/by-tmdb/:tmdbId` → zusätzlich `myProgress: { positionSec, durationSec, state: TitleState } | null`; `/nzb/shows/by-tmdb/:tmdbId` → zusätzlich `myProgress: { season, episode, positionSec, durationSec, state: TitleState }[]`.

- [ ] **Step 1: Write the failing tests:**
  - S1E3 fertig, Staffel 1 hat 8 Folgen, alle ausgestrahlt → Eintrag S1E4, `positionSec: 0`, ohne `NzbEpisode`-Zeile `nzbFileId: null`.
  - S1E8 fertig (letzte Folge) → S2E1; S2E1 mit `airDate` morgen → keine Serie in `items`.
  - S1E3 angefangen (`positionSec: 900`) → Eintrag S1E3 mit `positionSec: 900`.
  - Wiederholung: S1E5 fertig, danach S1E2 mit `PUT 120/3000` → Eintrag S1E2.
  - Nächste Folge hat `NzbEpisode` mit Stream-Datei → `nzbFileId` gesetzt; mit offener `EpisodeRequest` (`status: "searching"`) → `request.status === "searching"`.
  - `getTmdbSeasonById` liefert `{ status: "error" }` → Serie fehlt, Film-Einträge sind da, Status 200.
  - Film angefangen, Datei gelöscht (`nzbFileId` per SetNull `null`) → Eintrag bleibt, `nzbFileId: null`.
  - Film fertig → nicht in `items`. 21 angefangene Filme → 20 Einträge, neuester zuerst.
  - `DELETE /progress/show/:tmdbId` → Serie verschwindet; danach `PUT` auf eine Folge → Serie wieder da. `DELETE /progress/foo/1` → 400.
  - `GET /progress/next/:nzbFileId` für S1E3-Datei → `next.episode === 4`; für Film-Datei → `next: null`.
  - `/nzb/shows/by-tmdb` enthält `myProgress` mit `state: "started"` für S1E3.
- [ ] **Step 2: Run** `npx vitest --run src/routes/progress.test.ts` — FAIL.
- [ ] **Step 3: Implement** in `progress-service.ts`, `routes/progress.ts`, `nzb.ts`.
- [ ] **Step 4: Run** Tests — PASS; `npm test` und `npm run build` — grün.
- [ ] **Step 5: Commit** `feat(progress): continue-watching list, next episode and hide`

### Task 4: Web — Fortsetzen und Melden im Player

**Files:**
- Modify: `openmedia-web/src/lib/backend.ts` (Typen + Aufrufe)
- Create: `openmedia-web/src/lib/progress.ts` (Konstanten wie API `PROGRESS`, reine Helfer)
- Create: `openmedia-web/src/lib/progress-reporter.ts`
- Create: `openmedia-web/src/components/player/resume-prompt.tsx`
- Modify: `openmedia-web/src/components/movie/video-player.tsx`
- Modify: `openmedia-web/src/contexts/download-context.tsx` (`getStreamUrl` → `getStream`), alle Aufrufer (`movie-hero-with-player.tsx`, `show/season-list.tsx`, weitere per `grep getStreamUrl`)
- Test: `src/lib/progress.test.ts`, `src/lib/progress-reporter.test.ts`, `src/components/player/resume-prompt.test.tsx`

**Interfaces:**
- Consumes: API aus Task 2/3.
- Produces:
  - In `backend.ts`: `export interface Resume { positionSec: number; durationSec: number; finished: boolean }`; `DownloadLink` bekommt `resume: Resume | null`; `export function reportProgress(body: { nzbFileId: string; positionSec: number; durationSec: number }, token: string)`; `export function getContinueWatching(token: string)` → `{ items: ContinueItem[] }`; `export function getNextEpisode(nzbFileId: string, token: string)` → `{ next: NextEpisode | null }`; `export function hideContinue(kind: "movie" | "show", tmdbId: number, token: string)`; Typen `ContinueItem`, `NextEpisode` wie API Task 3.
  - In `download-context.tsx`: `getStream: (nzbFileId: string) => Promise<{ url: string; resume: Resume | null } | null>` ersetzt `getStreamUrl`.
  - In `progress.ts`: `export function shouldAskResume(resume: Resume | null): boolean`; `export function formatTimestamp(sec: number): string` (`2832` → `"47:12"`, `3723` → `"1:02:03"`).
  - In `progress-reporter.ts`: `export function createProgressReporter(send: (body: ProgressBody, beacon: boolean) => void, nzbFileId: string)` → `{ tick(positionSec: number, durationSec: number, playing: boolean): void; pause(...): void; seeked(...): void; ended(...): void; flush(beacon: boolean): void; dispose(): void }` — meldet höchstens alle 30 s beim Spielen, sofort bei Pause/Ende, nach Spulen erst nach 2 s Ruhe; meldet nie, solange `durationSec` nicht endlich und > 0 ist.
  - `VideoPlayer`-Props neu: `nzbFileId: string`, `resume?: Resume | null`, `startAt?: number` (setzt die Stelle ohne Frage). Beacon-Ziel: `navigator.sendBeacon("/api/backend/progress", new Blob([json], { type: "application/json" }))` (Methode `POST`).
  - `ResumePrompt` Props `{ positionSec: number; onResume(): void; onRestart(): void }`, Text „Weiterschauen ab {formatTimestamp}?“, Knöpfe „Fortsetzen“ und „Von vorne“.

- [ ] **Step 1: Write the failing tests:**
  - `shouldAskResume(null)` → `false`; `{2832, 6000, false}` → `true`; `{2832, 6000, true}` → `false`. `formatTimestamp` wie oben, `59` → `"0:59"`.
  - Reporter (fake timers): `tick` alle 1 s für 65 s → genau 2 Sendungen (bei 30 s und 60 s); `pause` → sofort; 5× `seeked` innerhalb 1 s → 1 Sendung nach 2 s; `flush(true)` → `send(..., true)`; `tick(10, NaN, true)` und `tick(10, 0, true)` → keine Sendung.
  - `ResumePrompt` zeigt „Weiterschauen ab 47:12?“; Klick „Fortsetzen“ ruft `onResume`, „Von vorne“ ruft `onRestart`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/progress.test.ts src/lib/progress-reporter.test.ts src/components/player` — FAIL.
- [ ] **Step 3: Implement** Helfer, Reporter, Prompt; im `VideoPlayer` Reporter an Vidstack-Ereignisse (`time-update`, `pause`, `seeked`, `ended`) hängen, `flush(true)` bei `pagehide`, `visibilitychange` → `hidden` und Unmount; Prompt vor dem Start, wenn `shouldAskResume(resume)` und kein `startAt`; bei „Fortsetzen“ `currentTime = resume.positionSec`. Aufrufer geben `nzbFileId` und `resume` weiter.
- [ ] **Step 4: Run** `npm test` und `npm run build` — grün.
- [ ] **Step 5: Commit** `feat(player): resume prompt and progress reporting`

### Task 5: Web — Nächste Folge (Knopf, Countdown, Download)

**Files:**
- Create: `openmedia-web/src/components/player/next-episode-overlay.tsx`
- Modify: `openmedia-web/src/lib/progress.ts`
- Modify: `openmedia-web/src/components/movie/video-player.tsx`
- Test: `src/lib/progress.test.ts`, `src/components/player/next-episode-overlay.test.tsx`

**Interfaces:**
- Consumes: `getNextEpisode`, `getStream` (Task 4), `requestEpisodes(body, token)` aus `backend.ts`.
- Produces:
  - `export type NextPhase = "hidden" | "button" | "countdown" | "download"`; `export function nextEpisodePhase(remainingSec: number, ended: boolean, next: NextEpisode | null): NextPhase` — `next` `null` → `hidden`; ohne `nzbFileId`: `download` ab `remainingSec <= 60` oder `ended`; mit `nzbFileId`: `countdown` bei `ended`, `button` bei `remainingSec <= 60`, sonst `hidden`.
  - `NextEpisodeOverlay` Props `{ phase: NextPhase; next: NextEpisode; onPlayNext(): void; onCancel(): void; onDownload(): void }`; Texte: „Nächste Folge ▶“, „Nächste Folge in {n} s“ mit „Jetzt abspielen“ / „Abbrechen“, „Nächste Folge herunterladen“; nach Download-Klick „Wird gesucht…“. Countdown zählt `PROGRESS.COUNTDOWN_SEC` herunter und ruft dann `onPlayNext`.
  - `VideoPlayer` holt `getNextEpisode(nzbFileId)` einmal beim Start (nur Folgen bekommen ein `next`), spielt `next` im selben Player weiter (neues `getStream`, neue `nzbFileId`, neuer Reporter, Titel `S{season} · F{episode} · {titleDe}`), Download über `requestEpisodes({ tmdbId: next.showTmdbId, season, episode })` ohne `guid`.

- [ ] **Step 1: Write the failing tests:**
  - `nextEpisodePhase(120, false, playable)` → `hidden`; `(60, false, playable)` → `button`; `(0, true, playable)` → `countdown`; `(30, false, notDownloaded)` → `download`; `(0, true, null)` → `hidden`.
  - Overlay (fake timers): `phase: "countdown"` zeigt „Nächste Folge in 10 s“, nach 10 s genau ein `onPlayNext`; „Abbrechen“ → `onCancel`, danach kein `onPlayNext`; `phase: "download"` Klick → `onDownload`, Text „Wird gesucht…“.
- [ ] **Step 2: Run** — FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `npm test`, `npm run build` — grün.
- [ ] **Step 5: Commit** `feat(player): next-episode button, countdown and download`

### Task 6: Web — Zeile „Weiterschauen“ und Fortschrittsbalken

**Files:**
- Create: `openmedia-web/src/components/home/continue-row.tsx`
- Modify: `openmedia-web/src/app/page.tsx`
- Modify: `openmedia-web/src/components/movie/movie-hero-with-player.tsx`, `openmedia-web/src/components/show/season-list.tsx` (Balken aus `myProgress`)
- Test: `src/components/home/continue-row.test.tsx`

**Interfaces:**
- Consumes: `getContinueWatching`, `hideContinue`, `getStream`, `requestEpisodes` (Task 4/5); `myProgress` (Task 3).
- Produces: `ContinueRow` (Client-Komponente, keine Props) über dem `MovieGrid`; rendert nichts bei leerer Liste oder ohne Login.
  - Kachel: Poster, Titel, bei Folgen „S2 · F5“, Balken `positionSec / durationSec` (bei `durationSec: 0` kein Balken), „×“ mit `aria-label="Aus Weiterschauen entfernen"` → `hideContinue` und Kachel sofort weg.
  - Klick mit `nzbFileId`: `getStream` und `VideoPlayer` als Vollbild-Overlay mit `startAt = resume?.positionSec` (keine Frage).
  - Folge ohne `nzbFileId`: Knopf „Herunterladen“ → `requestEpisodes`, danach Status-Text; mit `request` gleich der Status-Text wie in `season-list.tsx`.
  - Film ohne `nzbFileId`: Kachel verlinkt auf `/movie/{tmdbId}` (dort gibt es den bestehenden Ein-Klick-Download mit Rückfrage bei fremder Sprache).

- [ ] **Step 1: Write the failing tests** (Backend-Aufrufe gemockt): leere Liste → nichts im DOM; Film-Eintrag zeigt Titel und Balken mit 50 % bei `3000/6000`; Folge zeigt „S2 · F5“; „×“ ruft `hideContinue("show", tmdbId)` und entfernt die Kachel; Folge ohne Datei zeigt „Herunterladen“, Klick ruft `requestEpisodes` mit `{ tmdbId, season, episode }`; Film ohne Datei ist ein Link auf `/movie/{tmdbId}`.
- [ ] **Step 2: Run** `npx vitest --run src/components/home` — FAIL.
- [ ] **Step 3: Implement** Zeile, Einbau in `page.tsx`, Balken auf Detailseiten.
- [ ] **Step 4: Run** `npm test`, `npm run build` — grün. Lokal mit laufendem API im Browser prüfen: Film starten, nach 2 min schließen, Startseite zeigt Kachel, Klick spielt ab der Stelle.
- [ ] **Step 5: Commit** `feat(home): continue-watching row and progress bars`

### Task 7: tvOS — OpenMediaKit (Modelle, API-Client, reine Regeln)

**Files:**
- Modify: `openmedia-tvos/OpenMediaKit/Sources/OpenMediaKit/Models.swift`
- Modify: `openmedia-tvos/OpenMediaKit/Sources/OpenMediaKit/APIClient.swift`
- Create: `openmedia-tvos/OpenMediaKit/Sources/OpenMediaKit/Progress.swift`
- Test: `openmedia-tvos/OpenMediaKit/Tests/OpenMediaKitTests/ProgressTests.swift`, Ergänzung `APIClientTests.swift`

**Interfaces:**
- Consumes: API aus Task 2/3.
- Produces:
  - Modelle (`Codable, Sendable, Equatable`): `Resume { positionSec: Double; durationSec: Double; finished: Bool }`, `StreamLink { url: URL; resume: Resume? }`, `ProgressRequest { status: String; waitReason: String? }`, `NextEpisode { showTmdbId, season, episode: Int; titleDe: String; nzbFileId: String?; request: ProgressRequest? }`, `ContinueItem` mit den Feldern aus API Task 3 (`kind` als `enum ContinueKind: String { case movie, episode }`), `id` = `"\(kind)-\(tmdbId)"`.
  - `APIClient`: `func stream(nzbFileId: String) async throws -> StreamLink` (`streamURL` ruft es auf und bleibt für bestehende Aufrufer); `func reportProgress(nzbFileId: String, positionSec: Double, durationSec: Double) async throws` (`PUT /progress`); `func continueWatching() async throws -> [ContinueItem]`; `func nextEpisode(after nzbFileId: String) async throws -> NextEpisode?`; `func hideContinue(kind: String, tmdbId: Int) async throws` (`kind` `"movie"`/`"show"`).
  - `Progress.swift`: `public enum ProgressRules` mit denselben Zahlen wie API `PROGRESS`; `public func shouldAskResume(_ resume: Resume?) -> Bool`; `public func formatTimestamp(_ sec: Double) -> String`; `public enum NextPhase { case hidden, button, countdown, download }` und `public func nextEpisodePhase(remaining: Double, ended: Bool, next: NextEpisode?) -> NextPhase` (gleiche Regel wie Web Task 5); `public func shouldReport(positionSec: Double, durationSec: Double) -> Bool` (nur endlich, `durationSec > 0`).

- [ ] **Step 1: Write the failing tests:** gleiche Fälle wie Web für `shouldAskResume`, `formatTimestamp`, `nextEpisodePhase`; `shouldReport(10, .nan)` und `(10, 0)` → `false`. API-Client mit `StubURLProtocol`: `stream` dekodiert `resume` und `resume: null`; `reportProgress` sendet `PUT` an `/progress` mit Body `nzbFileId`, `positionSec`, `durationSec`; `continueWatching` dekodiert eine Folge mit `nzbFileId: null`; `nextEpisode` liefert `nil` bei `{"next": null}`; `hideContinue` sendet `DELETE /progress/show/95396`.
- [ ] **Step 2: Run** `cd openmedia-tvos/OpenMediaKit && swift test` — FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `swift test` — PASS.
- [ ] **Step 5: Commit** `feat(kit): progress models, API calls and rules`

### Task 8: tvOS — Player: Fortsetzen, Melden, nächste Folge

**Files:**
- Modify: `openmedia-tvos/OpenMedia/Player/PlayerView.swift`
- Modify: Aufrufer von `PlaybackTarget` (`MovieDetailView.swift`, `ShowDetailView.swift`, `LibraryView.swift`)

**Interfaces:**
- Consumes: alles aus Task 7.
- Produces: `PlaybackTarget { id: String; startAt: Double? }` (`startAt` = ohne Frage dorthin springen; bestehende Aufrufer `nil`).
  - Start: `api.stream(nzbFileId:)`; wenn `startAt == nil` und `shouldAskResume(resume)`: Dialog „Weiterschauen ab {formatTimestamp}?“ mit „Fortsetzen“ / „Von vorne“, dann `seek` + `play`.
  - Melden: `addPeriodicTimeObserver` alle `ProgressRules.reportIntervalSec`; zusätzlich bei Pause (`timeControlStatus`), in `onDisappear` und bei `AVPlayerItemDidPlayToEndTime`; nur wenn `shouldReport`. Fehler beim Melden werden geschluckt (Abspielen geht immer vor).
  - Nächste Folge: `nextEpisode(after:)` einmal beim Start; in den letzten 60 s `contextualActions = [UIAction(title: "Nächste Folge", …)]`; nach Datei-Ende Countdown-Ansicht über dem Player („Nächste Folge in {n} s“, „Jetzt abspielen“, „Abbrechen“); `download`-Phase: Knopf „Nächste Folge herunterladen“ → `requestEpisodes(tmdbId: next.showTmdbId, season:, episode:, guid: nil)`. Weiterspielen im selben `fullScreenCover`: alten `BufferedStream` schließen, neuen für `next.nzbFileId` öffnen, Melden für die neue Datei.

- [ ] **Step 1: Implement** wie oben (die Regeln selbst sind in Task 7 getestet; hier nur Verdrahtung).
- [ ] **Step 2: Build** `xcodebuild -scheme OpenMedia -destination 'generic/platform=tvOS Simulator' build` — Expected: `BUILD SUCCEEDED`.
- [ ] **Step 3: Manuell im Simulator oder auf `Wohnzimmer` prüfen** (Installation siehe Wiki `openmedia/openmedia-tvos-app.md`): Film 2 min spielen, Menü → zurück → neu starten zeigt „Weiterschauen ab 2:0x?“; Folge bis kurz vor Ende spulen → „Nächste Folge“ erscheint; Ende abwarten → Countdown, nach 10 s spielt die nächste Folge; Fortsetzen springt an die Stelle, die das Web gemeldet hat.
- [ ] **Step 4: Commit** `feat(tvos): resume, progress reporting and next-episode countdown`

### Task 9: tvOS — Zeile „Weiterschauen“ auf „Start“

**Files:**
- Modify: `openmedia-tvos/OpenMedia/Screens/HomeView.swift`
- Create: `openmedia-tvos/OpenMedia/Components/ContinueRow.swift`

**Interfaces:**
- Consumes: `continueWatching()`, `hideContinue`, `requestEpisodes`, `PlaybackTarget(id:startAt:)`, `MediaLink`.
- Produces: `ContinueRow` (waagerechte Reihe, nur wenn nicht leer) über „Trending Filme“; lädt beim Erscheinen von „Start“ neu.
  - Kachel wie Web (Poster, Titel, „S2 · F5“, Balken). Klick mit `nzbFileId` → `PlaybackTarget(id: nzbFileId, startAt: positionSec > 0 ? positionSec : nil)`.
  - Folge ohne Datei → Klick fordert an (`requestEpisodes`), danach Status-Text (`EpisodeRequest.label`-Texte wiederverwenden).
  - Film ohne Datei → `NavigationLink(value: MediaLink.movie(tmdbId))`.
  - Kontextmenü (lange drücken): „Aus Weiterschauen entfernen“ → `hideContinue`.

- [ ] **Step 1: Implement.**
- [ ] **Step 2: Build** wie Task 8 Step 2 — `BUILD SUCCEEDED`; `swift test` in `OpenMediaKit` — PASS.
- [ ] **Step 3: Manuell prüfen:** angefangener Film und Serie erscheinen; Entfernen per langem Drücken; nach fertiger Folge zeigt die Zeile die nächste.
- [ ] **Step 4: Commit** `feat(tvos): continue-watching row on start screen`

### Task 10: Auslieferung und Doku

**Files:**
- Modify: `llm-wiki/openmedia/openmedia-deployment.md` (Deploy-Historie), neuer Eintrag in `llm-wiki/log/`
- Modify: Status-Zeile der Spec auf „umgesetzt“

- [ ] **Step 1:** PRs für `openmedia-api`, `openmedia-web`, `openmedia-tvos` (Ziel `main`), CI grün, Review.
- [ ] **Step 2: Migration auf Prod** nach Wiki `openmedia/openmedia-deployment.md` („Migrations are never applied by the deploy“): `pg_dump -Fc` sichern, Builder-Image mit `--network openmedia_default` und `npx prisma migrate deploy`. Erwartet: `add_watch_progress` angewendet.
- [ ] **Step 3: API, dann Web deployen** (Rollback-Images `openmedia-*:pre-weiterschauen` taggen), `curl https://api.mediatoken.de/health` → 200, `https://mediatoken.de` lädt.
- [ ] **Step 4: tvOS** auf `Wohnzimmer` installieren (Wiki `openmedia/openmedia-tvos-app.md`).
- [ ] **Step 5: Live prüfen:** am TV stoppen, im Browser weiterschauen (und umgekehrt); Countdown in einer Serie; Zeile auf beiden Startseiten.
- [ ] **Step 6:** Wiki-Deploy-Historie + Log-Eintrag, `node scripts/build-index.mjs`, `node scripts/check-links.mjs`; Commit in `llm-wiki`.

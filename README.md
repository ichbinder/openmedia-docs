# OpenMedia

Persönliche Film-Plattform zum Entdecken, Verwalten und Streamen von Filmen — mit Usenet/NZB-Integration.

## Repositories

| Repo | Beschreibung | Tech Stack | Port |
|---|---|---|---|
| [openmedia-web](https://github.com/ichbinder/openmedia-web) | Frontend — Filme browsen, suchen, streamen | Next.js 16, shadcn/ui, Tailwind | 3000 |
| [openmedia-api](https://github.com/ichbinder/openmedia-api) | Backend — Auth, Watchlist, NZB-Datenbank | Express 5, Prisma, PostgreSQL | 4000 |
| [openmedia-nzb](https://github.com/ichbinder/openmedia-nzb) | NZB-Storage — Datei-Safe für NZB-Dateien | Express 5, Filesystem | 4100 |
| [openmedia-docs](https://github.com/ichbinder/openmedia-docs) | Dokumentation — Architektur, Diagramme | Markdown | — |

## Architektur-Übersicht

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
│                                                              │
│  ┌─────────────────────────┐    ┌──────────────────────────┐ │
│  │   openmedia-web (:3000) │    │  TMDB API (extern)       │ │
│  │   Next.js Frontend      │    │  Film-Metadaten + Bilder │ │
│  │                         │    └──────────┬───────────────┘ │
│  │  • Filme entdecken      │               │                 │
│  │  • Suche & Genres       │               │                 │
│  │  • Watchlist             │               │                 │
│  │  • Downloads             │               │                 │
│  │  • Bibliothek            │               │                 │
│  └──────────┬──────────────┘               │                 │
│             │ /api/backend/*               │                 │
│             │ (Proxy, httpOnly Cookie)     │                 │
│  ┌──────────▼──────────────────────────────▼───────────────┐ │
│  │            openmedia-api (:4000)                         │ │
│  │            Express Backend                               │ │
│  │                                                          │ │
│  │  • Auth (JWT + bcrypt)         • TMDB-Integration       │ │
│  │  • Watchlist CRUD              • NZB-Film-Datenbank     │ │
│  │  • NZB-Import Pipeline         • Download-Orchestrierung│ │
│  │                                                          │ │
│  │  ┌──────────────────────┐                                │ │
│  │  │  PostgreSQL (Docker) │                                │ │
│  │  │  • users              │                                │ │
│  │  │  • watchlist_items    │                                │ │
│  │  │  • nzb_movies         │                                │ │
│  │  │  • nzb_files          │                                │ │
│  │  └──────────────────────┘                                │ │
│  └──────────┬───────────────────────────────────────────────┘ │
└─────────────┼────────────────────────────────────────────────┘
              │ JWT (shared secret)
┌─────────────▼────────────────────────────────────────────────┐
│                     Lokales Netzwerk / NAS                     │
│                                                                │
│  ┌──────────────────────────┐                                  │
│  │  openmedia-nzb (:4100)  │                                  │
│  │  NZB Datei-Storage       │                                  │
│  │                          │                                  │
│  │  • PUT /files/:hash      │  ┌────────────────────────────┐ │
│  │  • GET /files/:hash      │  │  Filesystem                │ │
│  │  • DELETE /files/:hash   │──│  /data/nzb/                │ │
│  │  • GET /files (Liste)    │  │  ├── abc123...sha256.nzb   │ │
│  │  • HEAD /files/:hash     │  │  ├── def456...sha256.nzb   │ │
│  │                          │  │  └── ... (7256 Dateien)    │ │
│  └──────────────────────────┘  └────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────┐                                  │
│  │  Usenet-Downloader       │  (zukünftig: SABnzbd/NZBGet)    │
│  │  • NZB → Download        │                                  │
│  │  • Entpacken              │                                  │
│  │  • Bereitstellen          │                                  │
│  └──────────────────────────┘                                  │
└────────────────────────────────────────────────────────────────┘
```

## Ablaufdiagramme

### 1. User-Authentifizierung

```
Browser                   openmedia-web              openmedia-api           PostgreSQL
   │                          │                          │                      │
   │  POST /login             │                          │                      │
   │ (email, password)        │                          │                      │
   │─────────────────────────▶│                          │                      │
   │                          │  POST /auth/login        │                      │
   │                          │─────────────────────────▶│                      │
   │                          │                          │  SELECT user         │
   │                          │                          │─────────────────────▶│
   │                          │                          │  user row            │
   │                          │                          │◀─────────────────────│
   │                          │                          │  bcrypt.compare()    │
   │                          │  { user, token }         │                      │
   │                          │◀─────────────────────────│                      │
   │  Set-Cookie: token=JWT   │                          │                      │
   │  (httpOnly, secure)      │                          │                      │
   │◀─────────────────────────│                          │                      │
   │                          │                          │                      │
   │  GET /watchlist          │                          │                      │
   │  Cookie: token=JWT       │                          │                      │
   │─────────────────────────▶│                          │                      │
   │                          │  GET /watchlist           │                      │
   │                          │  Authorization: Bearer JWT│                     │
   │                          │─────────────────────────▶│                      │
   │                          │                          │  SELECT watchlist    │
   │                          │                          │─────────────────────▶│
```

### 2. NZB-Import Pipeline

```
NZB-Datei                 openmedia-api              TMDB API          PostgreSQL        openmedia-nzb
   │                          │                        │                  │                  │
   │  POST /nzb/import        │                        │                  │                  │
   │  (multipart/form-data)   │                        │                  │                  │
   │─────────────────────────▶│                        │                  │                  │
   │                          │  1. SHA-256 Hash       │                  │                  │
   │                          │  2. Duplikat-Check     │                  │                  │
   │                          │────────────────────────────────────────▶│                  │
   │                          │                        │   Kein Duplikat  │                  │
   │                          │◀───────────────────────────────────────│                  │
   │                          │  3. Dateiname parsen   │                  │                  │
   │                          │  (Titel, Jahr,         │                  │                  │
   │                          │   Auflösung, Sprache)  │                  │                  │
   │                          │                        │                  │                  │
   │                          │  4. TMDB-Suche         │                  │                  │
   │                          │───────────────────────▶│                  │                  │
   │                          │  { tmdbId, titleDe,    │                  │                  │
   │                          │    titleEn, year }     │                  │                  │
   │                          │◀───────────────────────│                  │                  │
   │                          │                        │                  │                  │
   │                          │  5. Movie upsert       │                  │                  │
   │                          │────────────────────────────────────────▶│                  │
   │                          │  6. NzbFile create     │                  │                  │
   │                          │────────────────────────────────────────▶│                  │
   │                          │                        │                  │                  │
   │                          │  7. NZB-Datei speichern│                  │                  │
   │                          │───────────────────────────────────────────────────────────▶│
   │                          │                        │                  │  PUT /files/:hash│
   │                          │                        │                  │  (NZB-XML)       │
   │                          │  { movie, nzbFile }    │                  │                  │
   │◀─────────────────────────│                        │                  │                  │
```

### 3. Film-Download (geplant)

```
User klickt                openmedia-api              openmedia-nzb        Usenet-Downloader
"Download 1080p"               │                          │                     │
   │                           │                          │                     │
   │  POST /downloads/start    │                          │                     │
   │  { movieId, nzbFileId }   │                          │                     │
   │──────────────────────────▶│                          │                     │
   │                           │  1. NzbFile.hash holen   │                     │
   │                           │  2. GET /files/:hash     │                     │
   │                           │─────────────────────────▶│                     │
   │                           │  NZB-XML Datei           │                     │
   │                           │◀─────────────────────────│                     │
   │                           │                          │                     │
   │                           │  3. NZB an Downloader    │                     │
   │                           │─────────────────────────────────────────────▶│
   │                           │                          │  Download startet   │
   │                           │  4. Status tracken       │                     │
   │                           │  (polling/webhook)       │                     │
   │                           │◀────────────────────────────────────────────│
   │  Status: "downloading"    │                          │                     │
   │◀──────────────────────────│                          │                     │
   │                           │                          │                     │
   │  ... Zeit vergeht ...     │                          │                     │
   │                           │                          │                     │
   │  Status: "ready"          │                          │  Download fertig    │
   │◀──────────────────────────│◀────────────────────────────────────────────│
```

## Datenmodell

### PostgreSQL (openmedia-api)

```
┌──────────────┐     ┌───────────────────┐     ┌──────────────────┐
│    users     │     │  watchlist_items   │     │   nzb_movies     │
├──────────────┤     ├───────────────────┤     ├──────────────────┤
│ id (uuid)    │◀───│ user_id (fk)      │     │ id (uuid)        │
│ email        │     │ movie_id (int)    │     │ tmdb_id (unique) │
│ name         │     │ title             │     │ imdb_id (unique) │
│ password     │     │ poster_path       │     │ title_de         │
│ created_at   │     │ vote_average      │     │ title_en         │
│ updated_at   │     │ release_date      │     │ description      │
└──────────────┘     │ added_at          │     │ year             │
                     └───────────────────┘     │ poster_path      │
                                               │ created_at       │
                                               │ updated_at       │
                                               └────────┬─────────┘
                                                        │ 1:n
                                               ┌────────▼─────────┐
                                               │   nzb_files      │
                                               ├──────────────────┤
                                               │ id (uuid)        │
                                               │ movie_id (fk)    │
                                               │ hash (unique)    │
                                               │ original_filename│
                                               │ file_size        │
                                               │ resolution       │
                                               │ audio_languages[]│
                                               │ subtitle_langs[] │
                                               │ codec            │
                                               │ source           │
                                               │ status           │
                                               │ broken_reason    │
                                               │ created_at       │
                                               │ updated_at       │
                                               └──────────────────┘
```

### Filesystem (openmedia-nzb)

```
/data/nzb/
├── 4532860a4bb9...sha256.nzb    ← The Matrix (1080p)
├── 7679bf039a48...sha256.nzb    ← 2-Headed Shark Attack (1080p)
├── ... (7256 NZB-Dateien, ~8 GB)
```

## Sicherheit

| Maßnahme | Beschreibung |
|---|---|
| **JWT + httpOnly Cookie** | Token wird nie im JavaScript sichtbar — XSS-geschützt |
| **bcrypt (12 Rounds)** | Passwörter werden gehasht gespeichert |
| **Proxy-Route** | Frontend spricht nie direkt mit Express — kein CORS nötig |
| **Path Traversal Protection** | NZB-Hashes werden auf Hex-Zeichen validiert |
| **Shared JWT Secret** | openmedia-api und openmedia-nzb nutzen den gleichen Schlüssel |

## CI/CD

Alle Repos nutzen GitHub Actions:

| Check | Trigger | Was |
|---|---|---|
| **Tests** | Push + PR | Vitest Unit/Integration Tests |
| **TypeScript** | Push + PR | `tsc --noEmit` Type-Check |
| **Hold-Check** | Label-Event | `hold` Label blockiert Auto-Merge |
| **CodeRabbit** | PR | AI Code-Review + Auto-Approve |
| **Greptile** | PR | Codebase-Context Review + Confidence Score |

### PR-Workflow

```
Feature-Branch → Push → PR erstellen → Auto-Merge aktivieren
    ↓
CI (Tests + TypeScript) → CodeRabbit Review → Greptile Review
    ↓
Greptile Score < 3/5? → hold Label setzen
    ↓
Fixes pushen → Re-Review
    ↓
CodeRabbit Approved + CI grün + hold-check grün → Auto-Merge
```

## Lokale Entwicklung

```bash
# 1. PostgreSQL starten
cd openmedia-api-project
docker compose up -d

# 2. Backend starten
cd server && npx tsx src/index.ts     # Port 4000

# 3. NZB-Storage starten
cd openmedia-nzb && npx tsx src/index.ts  # Port 4100

# 4. Frontend starten
npm run dev                            # Port 3000
```

## Aktuelle Statistiken

| Metrik | Wert |
|---|---|
| Filme in DB | 6.774 |
| NZB-Dateien | 7.256 |
| NZB-Speicher | ~8 GB |
| Backend-Tests | 58 |
| Frontend-Tests | 10 |
| E2E-Tests | 10 |

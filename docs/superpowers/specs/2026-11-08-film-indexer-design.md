# Film-Indexer in openmedia — Design

> Stand: 2026-11-08 · Status: Entwurf, wartet auf Freigabe

## 1. Absicht

Für Serien sucht openmedia die NZBs schon selbst bei Treasure Maps. Für Filme
geht das noch nicht. Der Film-Knopf funktioniert heute nur, wenn schon eine NZB
in openmedia liegt. Sonst zeigt er „Nicht verfügbar", und in der
Vorschau-Kachel verschwindet er ganz.

Der Betreiber will: **Knopf drücken, runterladen, fertig, gucken.** openmedia
nimmt dabei selbst die beste Version. Es ist egal, ob sie aus einer eigenen NZB
kommt oder von Treasure Maps. Es zählt nur, was gerade am besten passt. Wer
will, kann die Version auch selbst wählen. Das ist freiwillig und nicht der
normale Weg.

## 2. Ziele

1. Jeder Film-Knopf funktioniert auch ohne eigene NZB. openmedia sucht dann bei
   Treasure Maps, wählt die beste Version und startet den Download.
2. Eigene NZBs und Indexer-Treffer werden **gemeinsam** bewertet. Der Klick
   nimmt die beste von allen.
3. Der Nutzer kann über eine Liste selbst eine Version wählen. Die Liste zeigt
   eigene NZBs und Indexer-Treffer zusammen. Die beste steht oben. Auch wenn
   schon eine eigene NZB da ist, wird trotzdem beim Indexer gesucht.
4. Gibt es keine deutsche Version, dann bietet openmedia auch englische oder
   andere Sprachen an. Diese sind klar markiert.
5. Möglichst wenig Einstellungen für den Nutzer.

## 3. Nicht-Ziele

- **Kein automatischer Wechsel auf eine bessere Version.** Anders als bei
  Serien. Ein englisch geladener Film bleibt englisch. Erst wenn jemand wieder
  drückt, wird neu gewählt (Betreiber-Entscheidung).
- **Kein Abo, keine Merkliste mit Hintergrund-Suche** für Filme, die noch gar
  nicht erschienen sind. Im Hintergrund läuft nur das Weiterprobieren nach
  einem Fehlschlag (6.5).
- **Kein zweiter Indexer jetzt.** Die Anbindung bleibt aber so geschnitten wie
  bei Serien, damit später einer dazukommen kann.
- **Keine Änderung am Jellyfin-Plugin.** Filme erscheinen dort schon heute.
- **Kein neues Dashboard.** Nur der bestehende Knopf wird erweitert.

## 4. Anti-Ziele (das wäre ein Fehlschlag, auch wenn es technisch läuft)

- Ein Film ohne deutsche Tonspur wird geladen, ohne dass der Nutzer das vorher
  weiß und bestätigt hat.
- Ein Kino-Mitschnitt wird geladen, ohne dass der Nutzer das vorher weiß und
  bestätigt hat.
- Eine schlechte deutsche Version existiert, aber openmedia nimmt stattdessen
  eine gute englische.
- Der Knopf sagt „Verfügbar machen", aber der Klick führt ins Leere.
- Wiederholte Seitenaufrufe oder Kachel-Hover lösen jedes Mal eine neue
  Indexer-Suche aus.
- Bloßes Ansehen von Seiten verbraucht das Limit von 400 NZB-Downloads.
- Download-, Remux-, S3-, Archiv- oder Aufräum-Logik wird für Filme neu gebaut,
  statt die bestehende zu nutzen.

## 5. Ausgangslage (verifiziert im Code)

- **Repos:** `/Users/jschattka/git_private/cinescope/openmedia-*`, vor allem
  `openmedia-api` und `openmedia-web`.
- **Ein Knopf, zwei Orte:** `openmedia-web/src/components/movie/download-button.tsx`
  (`DownloadButton`, auch als `ProvisionButton` exportiert). Er wird benutzt in
  - der Film-Seite `/movie/[id]` über `components/movie/movie-actions.tsx`
  - der Vorschau-Kachel über `components/preview/movie-preview-actions.tsx`
    mit `hideWhenUnavailable`.
- **Heutige Knopf-Zustände:** „Anmelden zum Herunterladen", „Verfügbarkeit
  prüfen…", „Nicht verfügbar", grün „Verfügbar machen" (Icon `CloudDownload`
  aus lucide), gold „Herunterladen" (schon auf S3), sowie ein Dropdown mit den
  Gruppen „Bereit zum Herunterladen", „Verfügbar", „Fehlerhaft".
- **Daten:** Der Knopf holt die Versionen über `checkAvailability(tmdbId)` →
  `GET /nzb/movies/by-tmdb/:tmdbId` und startet über `startDownload(nzbFileId)`.
- **Indexer-Client:** `openmedia-api/src/lib/treasure-maps.ts` mit
  `searchEpisode`, `downloadNzb`, `getDownloadBudget`, Abstand 250 ms
  zwischen Anfragen und 1 h Such-Cache über `ttl-cache.ts`. Eine Film-Suche
  fehlt.
- **Auswahl-Regeln:** `openmedia-api/src/lib/release-ranking.ts`
  (`rankReleases`). Die Funktion filtert fest auf Staffel und Folge und wirft
  alles ohne deutsche Tonspur weg.
- **Ablauf bei Serien:** `openmedia-api/src/lib/episode-requests.ts`. Dort
  steht das Muster, das Filme übernehmen sollen: Kandidaten bewerten, als
  `broken` bekannte GUIDs ausfiltern, Budget prüfen unter `withBudgetLock`
  (Reserve 50), NZB laden, über `ingestNzb` anlegen, höchstens 3 Kandidaten
  probieren.
- **Ingest:** `openmedia-api/src/lib/nzb-ingest.ts` kann schon Filme
  (`source: "tmdb-found"; tmdb: TmdbMovieResult`).
- **Version selbst wählen bei Serien:** `GET /series/candidates` in
  `routes/series.ts` und `openmedia-web/src/components/show/episode-candidates.tsx`.
- **Treasure Maps für Filme** (`llm-wiki/web-sources/treasure-maps-indexer-api.md`):
  Film-Kategorien sind `2000` (international) und `2100 Movies - DE`.
  **Korrektur 2026-11-08:** Anders als `/tv` lehnt `GET /api/v1/movie` eine
  Suche nur mit `tmdbid` ab (`400 "Missing parameter: q or title"`), und mit
  `q` fallen Releases mit englischem Namen weg. Die Film-Suche nutzt darum
  `GET /api/v1/search?tmdbid=…&extended=true&limit=500` und behält nur die
  Kategorien 2000–2999.
- **Laufzeit:** `TmdbMovieResult` in `openmedia-api/src/lib/tmdb.ts` hat noch
  kein Feld `runtime`.

## 6. Entscheidungen des Betreibers

### 6.1 Ein Klick

- Der Klick lädt **sofort die beste Version**. Keine Auswahl, keine Frage —
  außer bei fehlender deutscher Tonspur (6.3).
- Eigene NZB oder Treasure Maps: openmedia nimmt, **was gerade am besten passt**.

### 6.2 Selbst wählen

- Es gibt immer die Möglichkeit, beim Indexer zu suchen, **auch wenn schon eine
  eigene NZB da ist**.
- Der Nutzer sieht dann die gefundenen Versionen und kann die Qualität selbst
  wählen.

### 6.3 Sprache

- Deutsche Tonspur gewinnt immer. Auch eine schlechte deutsche Version schlägt
  eine gute englische (gleiche Regeln wie bei Serien, Serien-Spec 6.4).
- Gibt es **keine** deutsche Version, dann zeigt openmedia englische oder
  andersprachige Versionen.
- Solche Versionen sind **eindeutig markiert**, am Knopf und in der Liste.
- Klickt jemand auf eine Version ohne deutschen Ton, dann fragt openmedia
  **erst kurz**, dann lädt es.

### 6.4 Später erscheint eine deutsche Version

- Nichts passiert von selbst. Es bleibt, wie es ist.

### 6.5 Fehlschlag nach dem Klick

- Scheitert ein Download erst später (zum Beispiel SABnzbd meldet Fehler,
  oder `ffprobe` zeigt, dass eine als deutsch gemeldete Version keinen
  deutschen Ton hat), dann **probiert openmedia selbst weiter**. Es nimmt die
  nächste Version, insgesamt höchstens 3, auch wenn niemand mehr auf der
  Seite ist.
- Dabei lädt openmedia **nie ungefragt** eine Version ohne deutschen Ton oder
  einen Kino-Mitschnitt. Bleibt nur noch so etwas übrig, dann hört es auf. Der
  Knopf zeigt danach wieder den passenden Zustand aus 6.7 mit der Frage.

### 6.6 Kino-Mitschnitte

- Abgefilmte oder schlechte Kino-Fassungen (zum Beispiel `CAM`, `HDCAM`,
  `TS`, `TELESYNC`, `TC`, Screener, `LD`/`MD`-Ton) werden **angezeigt, aber
  ganz unten**, sogar nach den Versionen ohne Deutsch.
- Sie haben ein Schild **„Kino-Mitschnitt"**. Vor dem Laden fragt openmedia
  kurz nach, wie bei „Nur Englisch".

### 6.7 Aussehen

Der Betreiber hat den Entwurf im Browser gesehen und angenommen:

1. **Normalfall:** „Suche Versionen…" → grün „Verfügbar machen · 1080p" → ein
   Klick → „Wird geladen · 34 %" → gold „Herunterladen · 1080p".
2. **Nur andere Sprache:** grüner Knopf mit gelbem Schild „Nur Englisch". Klick
   öffnet die Frage „Kein deutscher Ton — Diesen Film gibt es bisher nur auf
   Englisch. Trotzdem laden?" mit „Abbrechen" und „Englisch laden".
   Kino-Mitschnitte (6.6) sehen genauso aus, nur mit dem Schild
   „Kino-Mitschnitt" und einer passenden Frage.
3. **Nichts gefunden:** grauer Knopf „Keine Version gefunden".
4. **Selbst wählen:** kleiner Pfeil am Knopf öffnet eine Liste mit den Gruppen
   „Deutsch" und „Kein Deutsch". Jede Zeile zeigt Qualität, Release-Typ,
   Größe und Sprachen, zum Beispiel „1080p · WEB-DL · 7,8 GB" und darunter
   „Deutsch + Englisch · Untertitel DE · Empfohlen". Eigene NZBs stehen in
   derselben Liste, zum Beispiel „1080p · eigene NZB · a3f9c21e — schon im
   Bestand".

Hinweis des Betreibers: Das Wolken-Symbol im Entwurf war schlecht zu erkennen.
Im echten Code bleibt das lucide-Icon `CloudDownload`, das schon heute benutzt
wird. Es muss im Knopf gut erkennbar sein.

## 7. Eigene Festlegungen (noch nicht vom Betreiber bestätigt)

- **Suchen beim Öffnen:** Die Indexer-Suche läuft schon, wenn die Film-Seite
  öffnet oder die Kachel ihren Knopf zeigt, nicht erst beim Klick. Nur so kann
  der Knopf vorher „Nur Englisch" oder „Keine Version gefunden" zeigen.
  Suchen zählen nicht gegen das 400er-Limit, nur NZB-Downloads. Der
  bestehende 1-h-Cache je Film verhindert wiederholte Anfragen.
- **Gemeinsame Bewertung:** Eigene NZBs und Indexer-Treffer kommen in eine
  Liste und werden nach denselben Regeln sortiert. Eine Version, die schon auf
  S3 liegt, gewinnt in ihrer Sprachgruppe immer (siehe „Was gleiche Bewertung
  heißt"). Sonst gilt bei gleicher Bewertung: eigene NZB > Indexer-Treffer.
  Grund: Das spart Zeit und Limit. Eine eigene NZB, die gleichzeitig ein Indexer-Treffer ist (gleiche
  `indexerGuid`), erscheint nur einmal.
- **Schon fertig:** Liegt die beste Version schon auf S3, dann ist der Knopf
  gold „Herunterladen" wie heute.
- **Bewertung der Qualität:** dieselben Stufen wie bei Serien (1080p im
  Zielbereich → andere 1080p → 2160p → 720p → SD, dann Bonus für deutsche
  Untertitel und Hörfilm). Der Ziel-Größenbereich von 1,5–6 GB pro Stunde gilt
  auch für Filme. Laufzeit aus TMDB; fehlt sie, dann 110 Minuten.
- **Sprache als oberste Stufe:** Mit deutscher Tonspur steht immer vor ohne.
  Innerhalb von „ohne Deutsch" gilt dieselbe Qualitäts-Reihenfolge. Das
  Schild nennt die Sprache: „Nur Englisch", bei anderen Sprachen „Nur
  Französisch" usw., bei mehreren oder unbekannten Sprachen „Kein Deutsch".
- **Gefragt wird pro Klick.** Wer „Englisch laden" bestätigt, wird beim nächsten
  Film wieder gefragt. Es gibt keine Einstellung „nie mehr fragen".
- **Passwort-Releases** (`password = 2`) werden nie angezeigt, wie bei Serien.
- **Sprach-Prüfung nach dem Download:** Zeigt `ffprobe`, dass eine als deutsch
  gemeldete Version keinen deutschen Ton hat, dann wird sie `broken` wie bei
  Serien, und 6.5 greift. Das gilt nicht für Versionen, die der Nutzer bewusst
  ohne Deutsch geladen hat.
- **Weiterprobieren nur innerhalb derselben Gruppe:** Nach einem Fehlschlag
  nimmt openmedia nur Versionen, die nicht schlechter gruppiert sind als die
  ursprünglich gewählte (Deutsch → nur Deutsch; bestätigt Englisch → auch
  Englisch). Bewusst selbst gewählte Versionen (6.2) werden **nicht**
  automatisch ersetzt. Dann zeigt der Knopf den Fehler, und der Nutzer wählt
  neu.
- **Reihenfolge der Gruppen:** Deutsch → ohne Deutsch → Kino-Mitschnitt
  (innerhalb der Mitschnitte zuerst die deutschen). Innerhalb jeder Gruppe
  gelten die Qualitäts-Stufen.
- **Was „gleiche Bewertung" heißt:** gleiche Gruppe, gleiche Qualitäts-Stufe
  und gleicher Bonus (Untertitel/Hörfilm). Die Größe zählt dabei **nicht**.
- **Fertig gewinnt (Betreiber-Entscheidung 2026-11-08):** Innerhalb derselben
  Sprachgruppe steht eine Version, die schon auf S3 liegt, **vor** allen
  anderen, auch vor einer besseren Qualitäts-Stufe vom Indexer. Der Knopf
  zeigt dann gold „Herunterladen". Grund: Der Film ist schon da und lässt sich
  sofort schauen; kein Limit verbraucht. Deutsch schlägt weiter Nicht-Deutsch:
  Ein fertiger englischer Film verdrängt keine deutsche Version. Wer eine
  bessere Version will, wählt sie selbst (6.2).
- **Abgelaufene eigene NZBs** (`status = expired`, z. B. die MongoDB-Importe
  vom 2026-03-30) erscheinen gar nicht.
- **Keine harte Größengrenze.** Riesige Remuxe rutschen schon über den
  Ziel-Größenbereich nach hinten.
- **Alte Liste:** Die Gruppen „Deutsch", „Kein Deutsch" und „Kino-Mitschnitt"
  ersetzen „Bereit zum Herunterladen" und „Verfügbar". Der Zustand jeder
  Version steht als Schild in der Zeile („Bereit", „Wird geladen · 34 %",
  Abbrechen wie heute). Kaputte eigene NZBs bleiben wie heute ausgegraut unten
  in einer Gruppe „Fehlerhaft". Kaputte Indexer-Treffer werden ausgeblendet.
- **Wann gesucht wird:** nur für angemeldete Nutzer. Auf der Film-Seite beim
  Öffnen. In der Vorschau-Kachel erst, wenn die Kachel aufgeht (Hover), nicht
  für jedes Bild im Raster. Eine Suche pro Film, danach 1 h Cache.
- **Limit:** Filme und Serien teilen sich das Limit von 400 NZB-Downloads pro
  Tag mit derselben Reserve von 50. Ist das Limit erreicht, zeigt der Knopf
  „Tageslimit erreicht — morgen wieder" und lädt nichts.
- **Indexer gestört oder nicht eingerichtet:** Dann verhält sich der Knopf wie
  heute und zeigt nur die eigenen NZBs. Ohne eigene NZB: „Keine Version
  gefunden".
- **Vorschau-Kachel:** `hideWhenUnavailable` bedeutet ab jetzt „gar nichts
  gefunden, auch nicht beim Indexer". Während der Suche zeigt die Kachel den
  Knopf „Suche Versionen…".
- **Wer darf:** alle angemeldeten Nutzer, wie bei Serien.
- **Bibliothek:** Ein fertiger Film kommt in die `UserLibrary` des Nutzers, der
  geklickt hat, genau wie heute bei „Verfügbar machen".

## 8. Ablauf eines Klicks

1. Der Knopf erscheint. openmedia holt die eigenen NZBs und die Indexer-Treffer
   (aus dem Cache, wenn vorhanden) und bewertet alles gemeinsam.
2. Der Knopf zeigt den passenden Zustand aus 6.7.
3. Der Nutzer klickt. Ist die beste Version ohne deutschen Ton: erst die Frage.
4. Eigene NZB: der bestehende `startDownload`-Weg. Indexer-Treffer: Budget
   prüfen, NZB laden, über `ingestNzb` als `NzbFile` am Film anlegen. Dann
   übernimmt das bestehende `DownloadJob`-Verfahren (Provisionierung,
   SABnzbd, Remux, S3, Archiv).
5. Der Knopf zeigt den Fortschritt und wird gold, wenn der Film fertig ist.

## 9. Dem Umsetzer überlassen

- Neue Film-Suche in `treasure-maps.ts`. Umgesetzt über `/search` (siehe
  Abschnitt 5, Korrektur 2026-11-08).
- Wie `rankReleases` für Filme verallgemeinert wird (ohne Staffel/Folge-Filter,
  mit nicht-deutschen Treffern als eigene, niedrigere Gruppe), ohne das
  Verhalten für Serien zu ändern.
- Ob die Bewertung von eigenen NZBs und Treffern im API oder in der Webseite
  passiert. Naheliegend ist das API mit einem Endpunkt, der die fertig sortierte
  Liste liefert, ähnlich `GET /series/candidates`.
- Ob es einen eigenen Endpunkt „Film anfordern" gibt (wie
  `POST /series/requests`) oder ob der bestehende Weg erweitert wird.
- Wie das Weiterprobieren (6.5) sich merkt, welcher Nutzer angefordert hat,
  welche Gruppe erlaubt ist, ob die Version selbst gewählt war und wie viele
  Versuche schon liefen. Naheliegend: ein kleiner Film-Anforderungs-Eintrag
  und ein Haken im bestehenden `job-reconciler.ts`. Ein Warte-Status mit
  stündlicher Suche wie bei `EpisodeRequest` ist **nicht** nötig.
- Wie Kino-Mitschnitte erkannt werden (Namens-Hinweise, ggf. Indexer-Felder).
- `runtime` in `TmdbMovieResult` ergänzen.
- Woher die Sprachen eigener NZBs kommen (`NzbFile.audioLanguages`), und was
  bei eigenen NZBs ohne Sprachdaten gilt. Umgesetzt: mit ffprobe-Daten aus
  `audioLanguages`, sonst wie Indexer-Treffer ohne MediaInfo nur aus echten
  Namens-Hinweisen (der stille `en`-Standard des Parsers zählt als
  unbekannt).
- Genaue Gestaltung im Rahmen der bestehenden Komponenten und Farben.
- Tests im bestehenden Vitest-Stil, vor allem Tabellentests für die gemeinsame
  Bewertung (deutsch schlechter vs. englisch besser, eigene NZB vs. Treffer bei
  Gleichstand).

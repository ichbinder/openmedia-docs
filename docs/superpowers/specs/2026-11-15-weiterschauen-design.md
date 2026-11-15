# Weiterschauen für openmedia — Design

> Stand: 2026-11-15 · Status: umgesetzt und deployed (api 122bbc6, web 8f49a11, tvos 18629f8)
> Teil 1 von 4 des Vorhabens „KI-Vorschläge“ (siehe 1.1)

## 1. Absicht

Der Betreiber und seine Frau schauen abends auf der Apple-TV-App und auf der
Webseite. Sie stoppen oft mitten im Film oder mitten in einer Staffel, weil es
spät ist, und schauen ein, zwei Tage später weiter. Heute startet jeder Titel
wieder bei 0:00 und man muss von Hand hinspulen.

Dieses Projekt sorgt dafür, dass openmedia sich pro Konto merkt, wo man steht,
auf beiden Geräten gleich: am Fernseher stoppen, im Browser weiterschauen.

### 1.1 Einordnung

Das Gesamtvorhaben hat vier Teile, in dieser Reihenfolge:

1. **Weiterschauen** (dieses Dokument)
2. Herkunfts-Label (Netflix, HBO …) und bessere Bewertungen auf jeder Kachel
3. Eigene Bewertungen, Fragebogen, Trakt-Import
4. Vorschläge-Tab mit KI (OpenRouter)

Teil 3 und 4 lesen später die Daten, die Teil 1 sammelt (was wurde wie weit
geschaut, was wurde fertig). Deshalb muss Teil 1 die Daten sauber und pro
Titel speichern, auch wenn er sie selbst nur fürs Weiterschauen braucht.

## 2. Ziele

1. Beide Player (Web und tvOS) melden während des Abspielens regelmäßig die
   Stelle an die API. Auch beim Pausieren, beim Schließen und am Ende.
2. Beim Start eines Titels fragt der Player „Weiterschauen ab 47:12?“ mit
   **Fortsetzen** und **Von vorne**, wenn es eine passende Stelle gibt (siehe 6.1).
3. Eine Zeile **„Weiterschauen“** auf der Startseite, im Web und auf tvOS.
   Sie zeigt angefangene Filme und pro Serie die Folge, mit der es weitergeht.
4. Am Ende einer Folge: Countdown „Nächste Folge in 10 s“, dann spielt die
   nächste Folge automatisch (Entscheidung des Betreibers: Option B, wie Netflix).
   In den letzten 60 Sekunden steht schon ein Knopf „Nächste Folge ▶“.
5. Ist die nächste Folge nicht geladen, gibt es keinen Countdown, sondern den
   Knopf „Nächste Folge herunterladen“. In der Zeile „Weiterschauen“ erscheint
   die Folge mit dem bekannten Download-Knopf.
6. Der Fortschritt gehört zum **Titel** (Film bzw. Folge), nicht zur Datei.
   Eine andere Version desselben Films nimmt die Stelle mit.
7. Ein Eintrag lässt sich aus „Weiterschauen“ entfernen (Film doch nicht gut,
   Serie aufgegeben).

## 3. Nicht-Ziele

- **Keine Bewertungen, kein „Wie war es?“-Dialog**: kommt in Teil 3.
- **Keine Erkennung „abgebrochen“** und keine Nachfrage nach 30 Tagen: kommt in
  Teil 3/4. Teil 1 speichert nur die Rohdaten (Stelle, Länge, Zeitpunkte).
- **Kein Vorschläge-Tab**: die Zeile kommt jetzt nur auf die Startseite.
- **Kein Jellyfin, kein Trakt-Scrobble.** Der Betreiber schaut nicht über
  Jellyfin; Trakt-Regeln und Abhängigkeit sprechen dagegen.
- **Keine Abspann-Erkennung.** Es gibt keine Daten, wo der Abspann beginnt.
- Kein Verlauf aller Wiedergaben (jede einzelne Sitzung). Es zählt der letzte
  Stand pro Titel.

## 4. Anti-Ziele

- Eine Pause oder ein Abend-Stopp wird irgendwann als „mag ich nicht“ gelesen.
  Teil 1 speichert deshalb nichts, was nach Wertung aussieht, nur Fakten.
- Der Player springt beim Fortsetzen an eine falsche Stelle (andere Datei,
  andere Länge) und man merkt es erst nach Minuten.
- Das Melden der Stelle macht das Abspielen ruckelig oder spammt die API.
- Die zuletzt gesehene Stelle geht verloren, weil Tab oder App ohne letzte
  Meldung geschlossen wurde. Höchstens die letzten ~30 s dürfen fehlen.
- Der Countdown startet eine Folge, die gar nicht abspielbar ist (schwarzes Bild).

## 5. Rahmen (Ist-Zustand)

- **API** `openmedia-api`: Express 5, Prisma 7, PostgreSQL.
  Schema `prisma/schema.prisma`. Titel: `NzbMovie` (hat `tmdbId`),
  `NzbShow` (hat `tmdbId`) → `NzbEpisode` (`season`, `episode`).
  Versionen: `NzbFile` (gehört zu `movieId` **oder** `episodeId`, hat `duration`
  in Sekunden aus ffprobe, `s3StreamKey` = abspielbar). Ungenutzte Dateien
  löscht ein LRU-Aufräumer (`scheduledDeletionAt`).
- Abspielen startet immer mit `GET /nzb/files/:id/stream-link` (presigned URL).
- Serie mit Folgen und Versionen: `GET /nzb/shows/by-tmdb/:tmdbId`.
  Folgen anfordern: `POST /series/requests` (`lib/episode-requests.ts`).
  TMDB-Staffeldaten: `getTmdbSeasonById()` in `src/lib/tmdb.ts` (gecacht).
- **Web** `openmedia-web`: Next.js 16, Player = Vidstack
  (`src/components/movie/video-player.tsx`), genutzt von
  `movie-hero-with-player.tsx` (Film) und `show/season-list.tsx` (Folgen).
  Aufrufe zur API laufen über den Proxy `src/app/api/backend/[...path]/route.ts`.
  Startseite `src/app/page.tsx`: Hero + Trending-Grid.
- **tvOS** `openmedia-tvos`: SwiftUI, `OpenMedia/Player/PlayerView.swift`
  (`AVPlayerViewController`, `BufferedStream`), gestartet über
  `PlaybackTarget(id: nzbFileId)` aus Film-, Serien- und Bibliotheksseite.
  API-Client `OpenMediaKit/.../APIClient.swift`. Startseite `Screens/HomeView.swift`.
- Konten: ein Konto pro Haushalt. Weitere Konten (z. B. ein Freund) sind
  möglich; alles ist pro Konto, nichts ist auf eine Familie zugeschnitten.

## 6. Design

### 6.1 Regeln

| Regel | Wert |
|---|---|
| Meldeintervall beim Abspielen | alle 30 s, dazu bei Pause, Spulen-Ende, Schließen, Datei-Ende |
| „Fertig geschaut“ | Stelle ≥ 90 % der Länge, oder Datei-Ende erreicht |
| Frage „Fortsetzen?“ | nur wenn Stelle > 60 s **und** noch nicht fertig |
| Fortsetzen nach Fertig | Titel startet ohne Frage von vorne; „fertig“ bleibt gespeichert |
| „Nächste Folge ▶“-Knopf | in den letzten 60 s der Datei |
| Countdown | 10 s nach Datei-Ende, Knöpfe „Jetzt abspielen“ / „Abbrechen“ |
| Zwei Geräte gleichzeitig | letzte Meldung gewinnt |

Diese Zahlen stehen an **einer** Stelle pro Projekt als Konstanten, damit man
sie später leicht ändert.

**Zustand eines Titels** (für Zeile, Frage und nächste Folge), aus der
*aktuellen* Stelle abgeleitet:

- **angefangen**: `positionSec` > 60 s und < 90 % der Länge;
- sonst **fertig**, wenn `finishedAt` gesetzt ist;
- sonst **neu** (kaum angeschaut, erscheint nirgends).

Wer eine fertige Folge neu startet, ist also erst nach 60 s wieder „angefangen“.
`finishedAt` selbst bleibt als Fakt („einmal zu Ende gesehen“) für Teil 3/4.

### 6.2 Datenmodell

Neue Tabelle `watch_progress`, ein Eintrag pro Konto und Titel:

| Feld | Bedeutung |
|---|---|
| `userId` | Konto |
| `movieId` *oder* `episodeId` | genau eines ist gesetzt (Titel, nicht Datei) |
| `nzbFileId` | zuletzt benutzte Version, nullable, `onDelete: SetNull` (LRU darf löschen) |
| `positionSec` | letzte Stelle |
| `durationSec` | Länge der Datei, mit der zuletzt geschaut wurde |
| `finishedAt` | gesetzt, sobald einmal „fertig“; bleibt bei erneutem Schauen erhalten |
| `startedAt` | erste Meldung |
| `updatedAt` | letzte Meldung |
| `hiddenAt` | aus „Weiterschauen“ entfernt; wird bei neuer Wiedergabe wieder `null` |

Eindeutig pro (`userId`, `movieId`) und (`userId`, `episodeId`). Löschen des
Kontos oder Titels löscht den Fortschritt mit (Cascade). Diese Felder sind
bewusst die Rohdaten, die Teil 3/4 später auswerten.

**Fortsetzen mit anderer Version:** Die Stelle wird prozentual übertragen,
wenn die neue Datei mehr als 2 % länger oder kürzer ist
(`neueStelle = positionSec / durationSec × neueLänge`), sonst 1:1.
Dazu 5 s zurück, damit man den Anschluss sieht.

### 6.3 API

Neue Route `/progress` (requireAuth):

- `PUT /progress` — Body `{ nzbFileId, positionSec, durationSec }`.
  Die API bestimmt aus der Datei Film oder Folge, schreibt den Eintrag
  (upsert), setzt `finishedAt` nach 6.1, setzt `hiddenAt` zurück.
  Ungültige Werte (negativ, Position > Länge + Toleranz) → 400.
  Muss auch als `POST` mit gleichem Body gehen (für `navigator.sendBeacon`
  beim Schließen des Browser-Tabs). Antwortet immer mit JSON-Body (kein 204),
  weil der Web-Proxy jede Antwort als JSON liest.
- `GET /progress/continue` — die Zeile „Weiterschauen“, neueste zuerst, max. 20:
  - Film: Zustand „angefangen“, nicht versteckt.
  - Serie: **ein** Eintrag pro Serie, aus der Folge mit dem neuesten
    `updatedAt` („zuletzt geschaut“, auch bei einer Wiederholung):
    - versteckt → Serie erscheint nicht;
    - „angefangen“ → diese Folge mit Stelle;
    - „fertig“ → die nächste Folge nach TMDB-Reihenfolge (nächste Nummer in der
      Staffel, sonst Folge 1 der nächsten Staffel), nur wenn ihr `air_date`
      in der Vergangenheit liegt. Gibt es keine → Serie erscheint nicht.
  - Die nächste Folge hat meist noch **keine** `NzbEpisode`-Zeile (die entsteht
    erst bei einer Anfrage). Quelle ist deshalb `getTmdbSeasonById()` (gecacht).
    Ist TMDB nicht erreichbar, fällt nur dieser eine Serien-Eintrag weg; der
    Aufruf selbst schlägt nicht fehl.
  - Jeder Eintrag enthält: Art, TMDB-ID, Titel, Poster/Backdrop, bei Folgen
    Staffel/Folge und Folgentitel, Stelle, Länge, abspielbare `nzbFileId`
    (beste vorhandene Version mit `s3StreamKey`) **oder** `null` plus Status
    einer laufenden Anfrage:
    - Folge: gleiche Werte wie `myRequests` von `/nzb/shows/by-tmdb`;
    - Film: Status aus `MovieRequest` des Kontos.
  - **Datei weg, Titel angefangen** (z. B. vom LRU-Aufräumer gelöscht): Der
    Eintrag bleibt in der Zeile, mit Download-Knopf. Film → derselbe
    Ein-Klick-Weg wie auf der Detailseite (`POST /movies/requests`, beste
    Version); Folge → `POST /series/requests`. Die Stelle bleibt erhalten, weil
    sie am Titel hängt, und gilt nach dem neuen Download wieder (mit 6.2).
- `GET /progress/next/:nzbFileId` — für Countdown/Knopf am Folgen-Ende:
  nächste Folge (wie oben), mit abspielbarer `nzbFileId` oder `null`.
- `DELETE /progress/:kind/:tmdbId` — aus „Weiterschauen“ entfernen
  (setzt `hiddenAt` für den Film bzw. alle Folgen-Einträge der Serie). Das
  reicht auch für den Fall „nächste Folge“: dieser Eintrag leitet sich immer
  aus einer vorhandenen (fertigen) Folgen-Zeile ab, und die ist dann versteckt.
  Schaut man die Serie wieder, setzt `PUT` das `hiddenAt` der Folge zurück.
- `GET /nzb/files/:id/stream-link` liefert zusätzlich `resume`:
  `{ positionSec, durationSec, finished }` oder `null`. So braucht der Player
  beim Start keinen zweiten Aufruf.

„Beste vorhandene Version“ nutzt dieselbe Logik, die die Detailseiten heute
für „Abspielen“ nehmen. Gibt es dafür keine gemeinsame Funktion in der API,
wird sie hier angelegt und von beiden benutzt.

### 6.4 Web

- `video-player.tsx` bekommt `nzbFileId` und das `resume`-Objekt.
  - Frage „Weiterschauen ab mm:ss?“ als Overlay vor dem Start (Regeln 6.1).
  - Meldet nach 6.1; beim Schließen und bei `visibilitychange`/`pagehide`
    per `navigator.sendBeacon` an den Backend-Proxy. Der Body muss ein `Blob`
    vom Typ `application/json` sein; das JWT-Cookie geht same-origin mit.
  - Folgen: Knopf „Nächste Folge ▶“ in den letzten 60 s, Countdown am Ende,
    Daten aus `GET /progress/next/:nzbFileId`. Nächste Folge spielt im selben
    Player weiter (neue `stream-link`).
  - Nächste Folge nicht geladen: Knopf „Nächste Folge herunterladen“ →
    `POST /series/requests` mit `tmdbId`, `season`, `episode` (ohne `guid`,
    also beste Version wie bei einem normalen Klick).
- Startseite `src/app/page.tsx`: Zeile „Weiterschauen“ über dem Trending-Grid,
  waagerecht scrollbar, nur wenn sie nicht leer ist.
  - Kachel: Poster/Backdrop, Titel, bei Serien „S2 · F5“, Fortschrittsbalken.
  - Klick: spielt direkt an der Stelle (ohne Frage, man hat ja „Weiterschauen“ gewählt).
  - Ohne abspielbare Datei: bekannter Download-Knopf bzw. Status „wird geladen“.
  - Kleines „×“ zum Entfernen.
- Detailseiten zeigen bei angefangenen Titeln/Folgen einen Fortschrittsbalken
  (Film-Hero, Folgenliste). Klein halten, nur Anzeige.

### 6.5 tvOS

- `PlayerView` bekommt das `resume`-Objekt aus dem `stream-link`-Aufruf
  (`APIClient.streamURL` liefert es mit).
  - Frage „Fortsetzen / Von vorne“ vor dem Start, dann `seek` und `play`.
  - Melden über `addPeriodicTimeObserver` (30 s), bei Pause (`timeControlStatus`),
    in `onDisappear` und bei `AVPlayerItemDidPlayToEndTime`.
  - Folgen: „Nächste Folge“ über `AVPlayerViewController.contextualActions`
    (der Apple-Weg für genau diesen Knopf) in den letzten 60 s; Countdown nach
    Datei-Ende als eigene Ansicht über dem Player.
  - Nächste Folge spielt im selben `fullScreenCover` weiter (neuer `BufferedStream`,
    alten schließen).
- Startseite `HomeView`: Zeile „Weiterschauen“ über „Trending Filme“, gleiche
  Inhalte wie im Web. Entfernen per Kontextmenü (lange drücken).

## 7. Ansätze (betrachtet)

| Ansatz | Bewertung |
|---|---|
| **Fortschritt in der API, beide Player melden** | gewählt: geräteübergreifend, eine Logik für beide Apps, Daten für Teil 3/4 |
| Nur auf dem Gerät speichern (localStorage / UserDefaults) | verworfen: TV-Stopp ist im Browser unbekannt, keine Daten für die KI |
| Bei Trakt speichern (Scrobble) | verworfen: fremde Abhängigkeit, Trakt-App-Regeln (Piraterie-Klausel), Trakt kennt openmedia-Dateien nicht |

Ende einer Folge: **Countdown mit Autoplay** (Option B) statt nur Knopf
(Option A). Entscheidung des Betreibers: sie schlafen nicht vor der Serie ein.

## 8. Entscheidungen

**Vom Betreiber:**
- Weiterschauen in Web **und** tvOS, geräteübergreifend.
- Countdown mit automatischem Weiterspielen (Option B).
- Fehlt die nächste Folge: Download-Knopf, auch in „Weiterschauen“.
- Pause ist nie eine Wertung.
- Kein Jellyfin.

**Vorschläge des Designers (vom Betreiber nicht einzeln bestätigt):**
- Die Zahlen in 6.1 (30 s, 90 %, 60 s, 10 s).
- Fortschritt pro Titel statt pro Datei; prozentuale Übertragung bei anderer Länge.
- „Weiterschauen“-Zeile zuerst auf der Startseite.
- Eintrag entfernen per „×“ bzw. Kontextmenü.
- `resume` im `stream-link`-Ergebnis statt eigenem Aufruf.
- Angefangener Film ohne Datei bleibt in der Zeile, mit Download-Knopf
  (wie vom Betreiber für Folgen gewünscht). Umsetzung (Plan): Die Film-Kachel
  führt zur Detailseite, deren Ein-Klick-Download schon die Rückfrage bei
  fremder Sprache kann; Folgen-Kacheln fordern direkt an.
- „Zuletzt geschaut“ = neuestes `updatedAt`, auch bei Wiederholungen.
- Zustand „angefangen / fertig“ aus der aktuellen Stelle (Kasten in 6.1).
- **Kein Profil- oder Personen-Feld** in `watch_progress`: ein Konto pro
  Haushalt, der Betreiber mittelt den Geschmack bewusst. Die Zuordnung
  „Für wen?“ kommt in Teil 3 an die Bewertungen, nicht an den Fortschritt.

## 9. Dem Builder überlassen

- Genaue Prisma-Modellierung (z. B. Check-Constraint „genau eines von
  `movieId`/`episodeId`“ per SQL in der Migration).
- Aussehen von Overlay, Countdown und Kachel im Stil der bestehenden Komponenten.
- Ob das 30-s-Melden im Web über einen Hook oder direkt im Player sitzt.
- Drosselung beim Spulen (nicht bei jedem Sprung melden).
- Testaufteilung; Pflicht sind API-Tests für Upsert, „fertig“-Regel,
  nächste Folge (Staffelwechsel, nicht ausgestrahlt, fehlende Datei) und `continue`.

## 10. Auslieferung

API zuerst (enthält Migration), dann Web, dann tvOS (Installation per Xcode,
siehe Wiki `openmedia/openmedia-tvos-app.md`). Ältere Clients stören nicht:
sie melden einfach nichts.

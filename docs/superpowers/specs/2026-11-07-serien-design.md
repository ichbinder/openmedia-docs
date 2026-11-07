# Serien in openmedia — Design

> Stand: 2026-11-07 · Status: Entwurf, wartet auf Freigabe

## 1. Absicht

openmedia kann heute nur Filme. Nutzer (der Betreiber und seine Freunde) sollen
auch Serien finden, einzelne Folgen oder ganze Staffeln anfordern und sie
hauptsächlich auf dem Apple TV über Jellyfin schauen. Laufende Staffeln sollen
sich "abonnieren" lassen: Neue Folgen kommen von selbst, ohne dass jemand per
Hand NZBs sucht.

Heute sucht der Betreiber NZBs selbst und schickt sie per Browser-Erweiterung an
`POST /downloads/request`. Für Serien geht das nicht, weil Abos ohne Menschen
laufen müssen. Deshalb kommt mit den Serien eine Indexer-Anbindung dazu.

## 2. Ziele

1. Serien, Staffeln und Folgen werden in der Datenbank abgebildet, nach dem
   gleichen Versions-Muster wie Filme.
2. Auf der Webseite lassen sich Serien suchen und ansehen. Nutzer können dort
   Folgen oder Staffeln anfordern und Staffeln abonnieren.
3. openmedia sucht NZBs selbst beim Indexer Treasure Maps und wählt nach festen
   Regeln die beste Version aus.
4. Abonnierte Staffeln laden neue Folgen automatisch.
5. Erscheint später eine klar bessere Version, dann wechselt openmedia
   automatisch zu ihr.
6. In Jellyfin erscheinen Serien als echte Serien mit Staffeln und Folgen.
7. TMDB- und Indexer-Antworten werden zwischengespeichert, damit wiederholte
   Seitenaufrufe die API-Limits nicht belasten. Die Filme bekommen diesen Cache
   gleich mit.

## 3. Nicht-Ziele

- **Kein Kontingent pro Nutzer.** Der Speicher wird wie bei Filmen über die
  bestehende Aufräum-Logik verwaltet (`openmedia-api/src/lib/s3-lifecycle.ts`:
  ab 70 % von 1 TB werden die am längsten ungenutzten Dateien markiert und nach
  3 Tagen gelöscht, wenn niemand sie mehr in seiner `UserLibrary` hat).
- **Kein neues Dashboard- oder Vorschau-Konzept.** Die Serien-Startseite wird
  so einfach wie die Film-Startseite. Den Umbau von Dashboard und Vorschau
  plant der Betreiber als eigenes, späteres Projekt.
- **Kein echtes Video-Transkodieren.** Folgen bekommen denselben Remux wie Filme
  (Bild kopieren, Ton zu AAC Stereo). HEVC/x265 kann in manchen Browsern
  (außerhalb von Safari/Apple) nicht abspielbar sein. Das nehmen wir vorerst hin.
- **Kein Sonarr/Radarr, kein SSO.** Die alten Pläne in
  `mediatoken/.automaker/context/` sind überholt.
- **Kein zweiter Indexer jetzt.** Die Anbindung soll aber so geschnitten sein,
  dass ein zweiter Newznab-/REST-Indexer später leicht dazukommt.
- **Kein Anime-Sonderweg** (AniDB-IDs, absolute Nummerierung).

## 4. Anti-Ziele (das wäre ein Fehlschlag, auch wenn es technisch läuft)

- Eine Folge ohne deutsche Tonspur landet in der Bibliothek.
- Der Nutzer bekommt nichts angezeigt, obwohl eine schlechtere deutsche Version
  existiert.
- Beim Versionswechsel geht in Jellyfin der Gesehen-Status oder die
  Fortsetzen-Position verloren.
- Wiederholte Seitenaufrufe lösen jedes Mal neue TMDB- oder Indexer-Anfragen aus.
- Download-, Remux-, S3-, Archiv- oder Aufräum-Logik wird für Serien doppelt
  gebaut, statt die Film-Logik wiederzuverwenden.

## 5. Ausgangslage (verifiziert im Code)

- **Repos:** `/Users/jschattka/git_private/cinescope/openmedia-*`. Für die Serien
  wichtig sind `openmedia-api` (Express 5, Prisma 7, PostgreSQL),
  `openmedia-web` (Next.js 16.4), `openmedia-downloader` (SABnzbd + ffmpeg auf
  Hetzner-VPS bei Bedarf), `openmedia-uploader` (Re-Upload ins eigene Usenet)
  und `openmedia-jellyfin-plugin` (C#, Jellyfin 10.11).
- **Datenmodell Filme** (`openmedia-api/prisma/schema.prisma`): `NzbMovie` (1)
  → `NzbFile` (n Versionen). `NzbFile.hash` ist der SHA-256 der NZB und
  `@unique`. Dazu kommen Qualitätsfelder (`qualityTier`, `fileSize`,
  `audioLanguages`, `subtitleLanguages`, `codec`, `hdr`, ffprobe-Daten),
  S3-Felder (`s3Key`, `s3StreamKey`) und Lebenszyklus-Felder (`lastAccessedAt`,
  `scheduledDeletionAt`). `NzbFile.movieId` ist schon nullable.
- **Bibliothek:** `UserLibrary` verknüpft einen Nutzer mit einer konkreten
  Version (`nzbFileId`), nicht mit dem Film.
- **Download-Pipeline:** `DownloadJob` mit dem Ablauf `needs_review → queued →
  provisioning → downloading → uploading → completed | failed`.
  `openmedia-downloader/scripts/post-process.sh` nimmt **nur die größte
  Videodatei** (Zeile 96, `sort -rn | head -1`), macht daraus `{hash}_stream.mp4`
  (Zeile 200) und meldet an `PATCH /downloads/jobs/:id/status`.
- **Archiv:** `UploadJob` packt eine Datei von S3 und lädt sie über Nyuu ins
  Usenet. Das Ergebnis ist eine neue NZB mit eigenem Hash (`nzbHash`).
- **Jellyfin-Plugin:** `LibraryAutoCreateService.cs` legt eine Bibliothek vom Typ
  `CollectionTypeOptions.movies` an. `Tasks/StrmSyncEngine.cs` schreibt pro
  Film den Ordner `{Titel} ({Jahr}) [tmdbid-N]/{hash}.strm`. Die Daten kommen von
  `GET /jellyfin/library` und die Änderungserkennung von
  `GET /jellyfin/library/version`.
- **TMDB:** `openmedia-web/src/lib/tmdb.ts` nutzt ein normales `fetch()`
  (Zeile 112) **ohne Cache**. Jeder Seitenaufruf fragt TMDB neu. `openmedia-api/src/lib/tmdb.ts`
  kennt nur Film-Suchen.
- **Konfiguration:** Es gibt schon eine verschlüsselte Admin-Konfiguration
  (`EncryptedConfig`, `ConfigEntry`, `routes/admin-config.ts`). Dort liegen auch
  die Usenet-Zugangsdaten.

## 6. Entscheidungen des Betreibers

### 6.1 Anfordern

- Ein Nutzer sucht eine Serie und fordert **einzelne Folgen** oder **ganze
  Staffeln** an.
- Eine **laufende Staffel** lässt sich abonnieren. Jede neue Folge wird
  automatisch geladen, sobald sie erscheint.
- **Abo holt nach:** Beim Abonnieren werden die schon erschienenen Folgen der
  Staffel sofort mitgeladen.
- **Abo endet mit der Staffel:** Für die nächste Staffel muss neu abonniert
  werden. So wird nichts ungewollt geladen.
- **Alle Nutzer** dürfen abonnieren. Alle Abos teilen sich dasselbe Limit von
  400 NZB-Downloads pro Tag.

### 6.2 Anschauen

- Hauptweg ist **Apple TV über Jellyfin**. Die Serie muss dort als echte Serie mit
  Staffeln und Folgen erscheinen.
- **Fehlende Folgen werden nicht angezeigt.** Wer nur S02E03 anfordert, sieht in
  Staffel 2 nur diese eine Folge.
- Browser-Stream und Download sollen wie bei Filmen auch für Folgen gehen.

### 6.3 Indexer

- Angebunden wird **Treasure Maps** (`treasure-maps.com`) über die **REST API v1**.
  Alle Details stehen in
  `llm-wiki/web-sources/treasure-maps-indexer-api.md`. Das Wichtigste:
  - Authentifizierung per Header `X-API-Key`. Der Key kommt in die
    verschlüsselte Admin-Konfiguration und nie ins Repo.
  - Limits: 5 Anfragen pro Sekunde, kein Tageslimit für Suchen, aber **400
    NZB-Downloads pro 24 h**. `GET /api/v1/user` zeigt den aktuellen Verbrauch.
  - `GET /api/v1/tv?tmdbid=…&season=…&ep=…&cat=5100&extended=true` findet
    Folgen direkt über die TMDB-ID.
  - Jeder Treffer liefert vorab `audio_languages`, `subtitles`, `video.resolution`,
    `size`, `season`, `episode` und `password`.

### 6.4 Version auswählen

1. **Pflicht: deutsche Tonspur** (deutsche Synchronfassung). Treffer ohne
   deutsche Tonspur werden nie angezeigt und nie geladen.
2. **Immer etwas liefern:** Gibt es nur schlechte deutsche Versionen, dann wird
   trotzdem die beste davon angezeigt bzw. geladen.
3. **Reihenfolge:** 1080p im Ziel-Größenbereich → andere 1080p → 720p → SD.
   Die Größe sortiert nur, sie schließt nichts aus.
4. **Ziel-Größenbereich:** etwa 1,5 bis 6 GB pro Stunde Laufzeit (eigener
   Vorschlag, vom Betreiber angenommen). Die Werte sind in der Admin-Konfiguration
   änderbar. Innerhalb einer Stufe gilt: Größer ist besser, solange die Datei im
   Bereich liegt.
5. **Bonus, kein Muss:** deutsche Untertitel und eine Hörfilm-Tonspur
   (Audiodeskription). Sie geben bei gleicher Stufe Vorrang. Die Reihenfolge ist
   Deutsch + Untertitel + Hörfilm, dann Deutsch + Untertitel, dann nur Deutsch.
6. **Qualität sichtbar machen:** Jede Version zeigt dem Nutzer, was ihn erwartet,
   zum Beispiel "1080p · WEB-DL · 2,1 GB".

### 6.5 Speicherung und Versionen

- Gleiches Muster wie bei Filmen, nur eine Ebene tiefer: Serie → Folge →
  `NzbFile` (n Versionen, jeweils mit eigenem Hash).
- `NzbFile` bleibt **eine** Tabelle für Filme und Folgen. So gelten Download,
  Remux, S3, Archiv, Aufräumen und `UserLibrary` für beide.

### 6.6 Staffel-Pakete

- Gibt es eine Folge nicht einzeln, sondern nur im Staffel-Paket, dann wird
  **das ganze Paket** geladen. In der Bibliothek des Nutzers erscheint aber nur
  die angeforderte Folge.

### 6.7 Automatisch auf eine bessere Version wechseln

- Taucht später eine klar bessere Version auf, dann wechselt openmedia
  **automatisch**. Das gilt zum Beispiel für den Wechsel von SD auf 1080p.

### 6.8 Archiv und Remux

- Folgen kommen **wie Filme ins eigene Usenet-Archiv** (`openmedia-uploader`).
  Der Betreiber nimmt dafür das größere Datenvolumen in Kauf.
- Folgen bekommen denselben Remux zu `_stream.mp4` wie Filme. Ob die Last zu
  hoch wird, zeigt sich im Betrieb ("erstmal machen").

### 6.9 Dashboard und Cache

- Die Serien-Seiten sind zunächst so einfach wie die Film-Seiten: Trending oben
  als großes Bild, darunter ein Raster, dazu eine Detailseite und die Suche.
- **TMDB- und Indexer-Antworten werden zwischengespeichert**, damit
  wiederholte Besuche die API-Limits nicht belasten.

## 7. Eigene Festlegungen (vom Betreiber in der Zusammenfassung bestätigt)

Diese Punkte habe ich vorgeschlagen. Der Betreiber hat sie in der Zusammenfassung
gesehen und nicht widersprochen.

- **Jellyfin:** Das Plugin legt eine **zweite Bibliothek vom Typ Serien** an.
  Die Ordnerstruktur ist
  `{Serie} ({Jahr}) [tmdbid-N]/Season NN/{Serie} SNNENN.strm`.
- **Stabiler Dateiname:** Die `.strm`-Datei einer Folge heißt immer
  `… SNNENN.strm` und nie `{hash}.strm`. Beim Versionswechsel ändert sich nur
  der Link darin. So behält Jellyfin den Gesehen-Status. Filme bleiben
  unverändert.
- **Abo-Rhythmus:** höchstens einmal pro Stunde je offener Folge suchen.
- **Keine deutsche Fassung:** Die Folge bleibt auf "wartet" und openmedia sucht
  im Abo-Rhythmus weiter. Die Webseite zeigt "Noch keine deutsche Fassung".
- **Wechsel ohne Lücke:** Die neue Version wird erst komplett verarbeitet
  (Download, Remux, S3). Erst danach wird `UserLibrary` umgestellt. Die alte
  Version bleibt in der Datenbank, und die Aufräum-Logik entfernt sie später.
- **Kein Hin und Her:** Gewechselt wird nur auf eine höhere Stufe (SD → 720p →
  1080p → 1080p im Zielbereich). Eine etwas größere Datei derselben Stufe löst
  keinen Wechsel aus.
- **Ende der Suche:** Ist die Ziel-Stufe erreicht (1080p, deutsche Tonspur, im
  Größenbereich), dann hört die Suche auf. Sonst endet sie nach 30 Tagen. Die
  Zahl ist in der Admin-Konfiguration änderbar.
- **Wechsel überall:** Der Wechsel gilt für abonnierte Folgen und für einzeln
  angeforderte Folgen.
- **Staffel-Paket:** Jede Folge aus dem Paket wird **einzeln** verarbeitet, auf
  S3 gelegt und archiviert. Nur die angeforderte Folge kommt in die Bibliothek.
  Die anderen sind später sofort verfügbar oder werden vom Aufräumen entfernt.
- **Auflösung erkennen:** nach der **Bildbreite**, nicht nach der Höhe und nicht
  nach der Indexer-Kategorie. Breitbild-Releases sind beschnitten (1080p wird zu
  `1918x802`), und 1080p-Releases liegen beim Indexer in `5140 HD` statt in
  `5142 1080p`.
- **Sprache erkennen:** zuerst über `audio_languages`/`subtitles` vom Indexer.
  Fehlen diese Felder (es gibt doppelte Posts ohne MediaInfo), dann wird der
  Zwilling mit Daten bevorzugt. Als letzter Ausweg dienen Namens-Hinweise
  (`German`, `DL`). Nach dem Download prüft `ffprobe` die Werte. Erweist sich eine
  Version als nicht deutsch, dann wird sie als `broken` markiert und die nächste
  genommen.
- **Hörfilm erkennen:** nur über Namens-Hinweise oder MediaInfo-Titel der
  Tonspur. Wird ein Hörfilm nicht erkannt, ist das kein Fehler.
- **Cache-Dauer (Startwerte):** Trending und Listen 6 h, Serien-, Staffel- und
  Folgen-Infos 24 h, TMDB-Suche 1 h, Indexer-Suche je Folge 1 h.
- **Cache ohne neue Infrastruktur:** Next.js-Cache und/oder eine
  Datenbank-Tabelle. Kein Redis.

### 7.1 Festlegungen aus der Prüfung (noch nicht vom Betreiber gesehen)

Diese Punkte hat ein Prüfer als offen gemeldet. Ich habe sie selbst entschieden. Sie stehen hier zum Gegenlesen.

- **2160p/UHD:** Es zählt als eigene Stufe zwischen "andere 1080p" und "720p".
  Liefere also lieber 1080p, aber eine deutsche 2160p-Version ist besser als
  720p. Ein 2160p-Release erreicht nie die Ziel-Stufe und beendet die Suche
  deshalb nicht.
- **Mehrere Nutzer, eine Folge:** Beim automatischen Wechsel werden **alle
  aktiven** `UserLibrary`-Einträge dieser Folge umgestellt (`removedAt = null`).
  Ausgenommen sind Einträge, bei denen der Nutzer die Version **selbst
  ausgewählt** hat. Diese Einträge bleiben, wie sie sind. Dafür braucht
  `UserLibrary` ein Kennzeichen "manuell gewählt".
- **Pre-Cache:** Das bestehende Pre-Cache (`PrecacheRequest` pro
  `(userId, hash)`, lokale `.mp4` im Plugin) gilt auch für Folgen. Beim
  Wechsel wird ein offener oder fertiger Pre-Cache auf den alten Hash entfernt
  und für den neuen Hash neu angelegt. So zeigt Jellyfin nie die alte Datei,
  während die `.strm` schon die neue meint.
- **Staffel-Paket gegen Einzelfolge:** Pakete sind normale Kandidaten in der
  Auswahl (6.4). Die Größe pro Folge ist Paketgröße geteilt durch die Zahl der
  Folgen. Bei gleicher Stufe gewinnt die Einzelfolge, weil sie weniger lädt und
  verarbeitet. Ein Paket darf eine vorhandene Folge nur dann ersetzen, wenn es
  keine Einzelfolge in der höheren Stufe gibt.

## 8. Ablauf einer Folge

1. Ein Nutzer fordert eine Folge an. Das passiert direkt, über "Staffel anfordern"
   oder über ein Abo, wenn eine Folge laut TMDB erschienen ist.
2. openmedia sucht beim Indexer nach TMDB-ID, Staffel und Folge in `5100`.
   Gibt es keinen Einzeltreffer, dann sucht es das Staffel-Paket.
3. openmedia wählt die beste Version nach Abschnitt 6.4.
4. openmedia lädt die NZB (dieser Schritt zählt gegen das 400er-Limit) und legt
   `NzbFile` an. Das bestehende `DownloadJob`-Verfahren übernimmt dann
   Provisionierung, SABnzbd, Remux und S3.
5. Ist der Job fertig, dann kommt der Eintrag in die `UserLibrary` und es folgt
   ein `UploadJob` ins Archiv.
6. Das Plugin sieht die Änderung über `/jellyfin/library/version` und schreibt die
   `.strm`-Datei in die Serien-Bibliothek.
7. Solange die Ziel-Stufe nicht erreicht ist, sucht openmedia weiter (6.7 und 7).

## 9. Reihenfolge der Umsetzung (Vorschlag)

1. **Grundlage:** Datenmodell (Serie, Folge, `NzbFile` für beides), TMDB-TV im
   API, TMDB-Cache (auch für Filme), Serien-Seiten auf der Webseite, Serien-Bibliothek
   im Jellyfin-Plugin. Für den Test reicht vorerst der manuelle NZB-Weg über die
   Erweiterung.
2. **Indexer:** Treasure-Maps-Client, Auswahl-Regeln, Versionsliste auf der
   Detailseite, "Folge/Staffel anfordern" ohne manuelle NZB.
3. **Staffel-Pakete:** Mehrere Dateien pro Download verarbeiten (Downloader und
   Callback).
4. **Abo und Wechsel:** Abo pro Staffel, stündlicher Abgleich, automatischer
   Wechsel auf die bessere Version, Ende nach Ziel-Stufe oder 30 Tagen.

Jede Stufe ist für sich nutzbar.

## 10. Dem Umsetzer überlassen

- Genaue Prisma-Modelle und Namen, zum Beispiel ob es eine eigene
  Staffel-Tabelle gibt und wie `NzbFile` auf Film oder Folge zeigt (zwei
  nullable Fremdschlüssel mit Check-Constraint oder Ähnliches). Dazu die
  Migration, ohne bestehende Film-Daten zu verändern.
- Wo die Abo-Logik läuft. Naheliegend ist der bestehende `job-reconciler.ts`.
- Wie der Downloader bei Staffel-Paketen mehrere Dateien zurückmeldet und wie
  die Hashes der Folgen-Versionen aus einem Paket gebildet werden, bis der
  Archiv-Upload eigene NZBs liefert.
- Wie auf Indexer-Fehler (HTTP 429) und auf das Download-Limit reagiert wird,
  zum Beispiel Drosselung oder Warteschlange. Ziel ist, dass ein Abo nie das
  Tageslimit aufbraucht.
- Woher "Folge ist erschienen" kommt: TMDB-`air_date` oder der Indexer selbst.
- Wie genau `/jellyfin/library` Folgen ausliefert und wie das Plugin die zweite
  Bibliothek anlegt und befüllt. Bestehende Film-Bibliotheken bei Nutzern dürfen
  dabei nicht kaputtgehen.
- Gestaltung der Serien-Seiten im Rahmen der bestehenden Film-Komponenten.
- Tests im bestehenden Vitest-Stil. Für die Auswahl-Regeln (6.4) gibt es
  Tabellentests mit echten Release-Daten aus dem Wiki-Beispiel (Severance S02E05).

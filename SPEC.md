# Anforderungsspezifikation: Pferde-Gesundheits-App (iOS/iPadOS)

Stand: 9. November 2026

## 1. Zweck und Produktvision

Ziel ist eine native iPhone- und iPad-App, die für Pferde das leistet, was Apple Health für Menschen leistet: Messwerte, Fütterung, Symptome und Behandlungen pro Pferd erfassen, im Zeitverlauf darstellen und an fällige Termine erinnern. Arbeitstitel: **EquiHealth** (Platzhalter, später änderbar).

**Zielgruppe:** Pferdebesitzer und Reitbeteiligungen mit einem oder mehreren Pferden, überwiegend in Deutschland.

**Hinweise für Claude Code**

- Dieses Dokument ist die verbindliche Quelle für Umfang und Verhalten. Lege es als `SPEC.md` ins Repository-Root und verweise in `CLAUDE.md` darauf.
- Arbeite die Phasen aus Abschnitt 7 nacheinander ab. Jede Phase endet mit lauffähigem Build, grünen Tests und erfüllten Akzeptanzkriterien.
- Anforderungen sind mit IDs versehen (z. B. `F-MESS-01`). Verweise in Commits und Tests auf diese IDs.
- **MUSS** = Pflicht für Version 1.0, **SOLL** = gewünscht, darf verschoben werden, **KANN** = optional bzw. spätere Version.
- Bei Unklarheiten: Annahme treffen, im Code als `// ANNAHME:` kommentieren und in `OPEN_QUESTIONS.md` notieren, statt die Arbeit anzuhalten.
- Die App ist ein Tagebuch- und Erinnerungswerkzeug, kein Diagnosewerkzeug. Sie gibt keine medizinischen Empfehlungen.

## 2. Technische Rahmenbedingungen

Die App wird vollständig mit Apple-Frameworks gebaut, ohne Fremdbibliotheken in Version 1.0. Die einzige externe Abhängigkeit ist die Wetter-API.

| Bereich | Festlegung |
| --- | --- |
| Plattformen | iPhone und iPad (Universal App), Mindestversion iOS/iPadOS 27; auf Apple-Silicon-Macs als „Designed for iPad“, keine eigene Mac-Version |
| Sprache | Immer die neueste stabile Swift-Version (Stand November 2026: Swift 6.4 mit Xcode 27), Swift-6-Sprachmodus mit Strict Concurrency |
| UI | SwiftUI, aktuelles Systemdesign (Liquid Glass), keine eigenen Designsysteme |
| Persistenz | SwiftData |
| Synchronisierung | SwiftData mit CloudKit (private Datenbank): vorbereitet, aber per Feature-Flag aus, bis ein Entwicklerkonto vorhanden ist |
| Diagramme | Swift Charts |
| Erinnerungen | UserNotifications (lokale Benachrichtigungen) |
| Standort für Wetter | CoreLocation (einmalig beim Anlegen eines Stalls) und MapKit-Suche |
| Wetterdaten | [Bright Sky](https://brightsky.dev/) (JSON-API für die offenen DWD-Daten, ohne API-Schlüssel) |
| Tests | Swift Testing für Logik, XCUITest für Kernabläufe |
| Lokalisierung | String Catalogs, Deutsch als Hauptsprache, Englisch vorbereitet |

**Architektur**

- `F-ARCH-01` MUSS: Datenmodell, Export-Formate und die Trainings-Schnittstelle liegen in einem lokalen Swift Package `EquiCore`. So kann die spätere Trainings-App dasselbe Paket einbinden.
- `F-ARCH-02` MUSS: Feature-Ordner pro Bereich (Pferde, Messwerte, Fütterung, Gesundheit, Behandlungen, Erinnerungen, Wetter, Export, Einstellungen).
- `F-ARCH-03` MUSS: Netzwerk- und Wetterlogik hinter einem Protokoll (`WeatherProvider`), damit der Anbieter austauschbar und testbar ist.
- `F-ARCH-04` MUSS: Alle Einheiten intern in SI-Einheiten speichern (kg, cm, °C), Umrechnung nur in der Anzeige über `Measurement`.

**Systemkontext (Zielbild mit Entwicklerkonto)**

```
┌──────────────────────────┐   Deep Links   ┌──────────────────────────┐
│ EquiHealth (diese App)   │ ◄────────────► │ Trainings-App (geplant)  │
│ Pferde, Messwerte,       │                │ startet und zeichnet     │
│ Gesundheit, Termine      │                │ Einheiten auf            │
│ bindet EquiCore ein      │                │ bindet EquiCore ein      │
└──────┬─────────────┬─────┘                └────────────┬─────────────┘
       │ Wetter      │ Sync                              │ Sync
       ▼             ▼                                   ▼
┌─────────────┐  ┌──────────────────────────────────────────────────────┐
│ Bright Sky  │  │ iCloud (CloudKit, privat, gemeinsamer Container)     │
│ DWD-Wetter  │  │ EquiHealth schreibt Pferde und Gesundheit,           │
│ gerundete   │  │ Trainings-App schreibt Einheiten                     │
│ Koordinaten │  └──────────────────────────────────────────────────────┘
└─────────────┘
```

Die Apps sprechen nie direkt miteinander über Daten. Sie lesen und schreiben im selben iCloud-Container und öffnen sich gegenseitig nur per Deep Link.

**CloudKit-Regeln für das Datenmodell** (sonst schlägt die Synchronisierung fehl):

- Jede Eigenschaft ist optional oder hat einen Standardwert.
- Keine `@Attribute(.unique)`-Constraints.
- Alle Beziehungen sind optional und haben eine definierte Inverse.
- Schemaänderungen nach Veröffentlichung nur additiv, über `VersionedSchema` und Migrationsplan.
- Große Binärdaten (Fotos, Befunde) mit `@Attribute(.externalStorage)`.

**Betrieb ohne Entwicklerkonto**

Derzeit gibt es kein bezahltes Apple-Entwicklerkonto. Die App muss trotzdem vollständig lauffähig sein.

- `F-ARCH-05` MUSS: Die App läuft im Simulator und auf einem eigenen Gerät mit kostenlosem Apple-Account (Personal Team). Sie nutzt keine Funktion, die ein bezahltes Konto voraussetzt, insbesondere iCloud/CloudKit und App Groups.
- `F-ARCH-06` MUSS: Zentrale Feature-Flags in `EquiCore` (`FeatureFlags`): `cloudSyncEnabled`, `sharedTrainingContainerEnabled`, `sharingEnabled`, `widgetsEnabled`. Alle stehen auf `false`. Bei `false` nutzt SwiftData einen rein lokalen Store (`cloudKitDatabase: .none`).
- `F-ARCH-07` MUSS: Entitlements für iCloud und App Group liegen als vorbereitete Dateien im Projekt, sind aber in keinem Target eingebunden. Die Anleitung zum Aktivieren steht in `ENABLE_CLOUDKIT.md`.
- `F-ARCH-08` MUSS: Das Schema bleibt trotzdem CloudKit-konform. So lässt sich der Sync später ohne Datenmigration einschalten.
- Hinweis: Mit kostenlosem Account läuft die Installation auf dem iPhone nach 7 Tagen ab und muss aus Xcode neu aufgespielt werden. Die Daten bleiben erhalten, solange die App nicht gelöscht wird. Die Backup-Funktion (4.9) ist deshalb früh wichtig.
- Ohne Konto funktionieren: lokale Mitteilungen, Wetter, CSV-Export, Backup und Deep Links.

## 3. Datenmodell

Das Pferd ist die zentrale Entität: Fast jeder Eintrag gehört genau zu einem Pferd. Jede Entität hat zusätzlich `id: UUID`, `createdAt` und `updatedAt`. Typen mit festen Auswahlwerten werden als Swift-`enum` mit `String`-Rohwert gespeichert, damit Export und Migration stabil bleiben.

| Entität | Zweck | Wichtige Felder | Beziehungen |
| --- | --- | --- | --- |
| `Horse` | Ein verwaltetes Pferd | Name, Rufname, Geburtsdatum, Rasse, Geschlecht (Stute/Wallach/Hengst), Farbe, Abzeichen, UELN, Chipnummer, Equidenpass-Status (Schlachttier ja/nein), Foto, Notizen, archiviert, Sortierindex | hat alle Einträge unten |
| `Stable` | Standort für Wetterabfragen | Name, Ort, Breitengrad, Längengrad | über `StableAssignment` mit Pferden verbunden |
| `StableAssignment` | Welches Pferd wann wo stand | gültig ab, gültig bis (leer = aktuell) | Horse, Stable |
| `Measurement` | Ein Messwert | Typ (Gewicht, Stockmaß, Bauchumfang, Brustumfang, Körperlänge, Körpertemperatur; erweiterbar um Puls, Atemfrequenz), Wert, Zeitpunkt, Methode (Waage, Maßband, Schätzformel, Thermometer), Notiz | Horse |
| `Feed` | Ein Futtermittel (Stammdaten) | Name, Kategorie (Raufutter, Kraftfutter, Mineralfutter, Ergänzungsfutter, Saftfutter, Sonstiges), Hersteller, Standardeinheit, archiviert | genutzt von Futterplan und Futtereintrag |
| `FeedingPlanItem` | Reguläre Ration mit Gültigkeit | Menge, Einheit, Mahlzeit (morgens/mittags/abends/ad libitum), gültig ab, gültig bis | Horse, Feed |
| `FeedingLog` | Abweichende oder einmalige Fütterung | Menge, Einheit, Zeitpunkt, Notiz | Horse, Feed |
| `SymptomEntry` | Beobachtetes Symptom | Kategorie (Husten, Spannigkeit, Kolik, Augenentzündung, Lahmheit, Durchfall, Hautveränderung, Fieber, Sonstiges), Schweregrad (leicht/mittel/schwer), Beginn, Ende (leer = laufend), typspezifische Details, Fotos, Notiz | Horse, optional Condition und Treatment |
| `Condition` | Krankheit oder Diagnose | Name, diagnostiziert am, ausgeheilt am, chronisch ja/nein, diagnostiziert von, Notiz | Horse, Symptome, Behandlungen |
| `Treatment` | Termin oder Behandlung | Art (Tierarzt, Impfung, Zahnbehandlung, Hufschmied, Osteopathie, Physiotherapie, Wurmkur, Sonstige), Datum, Beschreibung, Kosten (KANN), Anhänge | Horse, Contact, Medications, Condition |
| `Medication` | Verabreichtes Medikament oder Impfstoff | Präparat, Wirkstoff, Dosis, Einheit, Verabreichungsweg, Beginn, Ende, Häufigkeit, Chargennummer, Wartezeit Lebensmittel bis, Karenzzeit Turnier bis, Impfung gegen (Influenza, Tetanus, Herpes, Tollwut, Sonstige) | Treatment |
| `Contact` | Tierarzt, Hufschmied, Therapeut | Name, Rolle, Praxis, Telefon, E-Mail, Adresse | Behandlungen |
| `ReminderRule` | Wiederkehrende Fälligkeit | Art (z. B. Impfung Influenza, Zahnkontrolle, Hufschmied, Wurmkur, frei), Intervall (Zahl + Tage/Wochen/Monate/Jahre), Vorlaufzeit, Uhrzeit der Erinnerung, aktiv | Horse, Bezug zur passenden Treatment-Art |
| `Attachment` | Foto oder PDF (Befund, Rechnung, Equidenpass-Seite) | Datei (externer Speicher), Dateityp, Titel | SymptomEntry, Treatment, Horse |
| `WeatherDay` | Lokal gecachte Tageswetterdaten | Datum, gerundete Koordinaten, Temperatur min/mittel/max, Niederschlag, Luftfeuchte, Wind, Wetterlage, Station, abgerufen am | keine; **wird nicht über iCloud synchronisiert** (eigener lokaler Store) |
| `TrainingSession` | Von der Trainings-App geschriebene Einheit, in dieser App nur lesbar | Typ, Beginn, Ende, Dauer, Distanz, Zeit pro Gangart, Herzfrequenz Ø/max, Route, Quell-App, Notiz | Horse |

**Typspezifische Symptom-Details** (als Codable-Struct im `SymptomEntry`):

- Husten: trocken/feucht, Nasenausfluss (ja/nein, Farbe), Situation (Ruhe, Arbeit, Fütterung)
- Spannigkeit: betroffene Region, nach Arbeit ja/nein, Muskulatur hart ja/nein
- Kolik: Dauer, Anzeichen (Scharren, Wälzen, Schwitzen, kein Kotabsatz), Tierarzt gerufen ja/nein
- Augenentzündung: Auge (links/rechts/beide), Tränenfluss, Schwellung, Lichtempfindlichkeit

**Fälligkeit einer Erinnerung:** nächster Termin = Datum der letzten passenden Behandlung + Intervall. Gibt es noch keine, gilt ein vom Nutzer gesetztes Startdatum.

## 4. Funktionale Anforderungen

Zwölf Funktionsbereiche, jeweils mit nummerierten Anforderungen. Alles ohne Kennzeichnung ist MUSS.

### 4.1 Pferdeverwaltung

- `F-PFERD-01`: Beliebig viele Pferde anlegen, bearbeiten, archivieren und endgültig löschen (mit Bestätigung, löscht alle zugehörigen Einträge).
- `F-PFERD-02`: Ein Pferd ist immer „aktiv ausgewählt“. Der Wechsel erfolgt über ein Menü in der Navigationsleiste. Die Auswahl bleibt nach Neustart erhalten.
- `F-PFERD-03`: Profilseite pro Pferd mit Foto, Stammdaten, Alter (berechnet), aktuellem Stall und den jüngsten Werten.
- `F-PFERD-04`: Stall zuordnen mit Zeitraum. Ein Umzug beendet die alte Zuordnung und beginnt eine neue.
- `F-PFERD-05` SOLL: Reihenfolge der Pferde per Drag-and-drop ändern.

### 4.2 Messwerte

- `F-MESS-01`: Erfassen von Gewicht (kg), Stockmaß (cm), Bauchumfang (cm), Brustumfang (cm), Körperlänge (cm) und Körpertemperatur (°C) mit Datum, Uhrzeit, Methode und Notiz.
- `F-MESS-02`: Schnelleingabe mit wenigen Taps: Wert, Zeitpunkt vorbelegt mit jetzt, Speichern.
- `F-MESS-03`: Detailansicht pro Messtyp mit Swift-Charts-Diagramm und Zeiträumen T / W / M / 6M / J / Alle, wie in Apple Health. Darunter die Liste aller Einträge, bearbeitbar und löschbar per Wischgeste.
- `F-MESS-04`: Pro Messtyp ein einstellbarer Referenzbereich, im Diagramm als Band dargestellt. Der Wert ist eine Nutzereinstellung, kein medizinischer Grenzwert. Werte außerhalb werden farbig markiert.
- `F-MESS-05`: Bei Körpertemperatur optional die Tagestemperatur der Luft als zweite Linie einblenden (siehe 4.7).
- `F-MESS-06` KANN: Weitere Messtypen (Puls, Atemfrequenz, Body Condition Score) über dasselbe Modell.

**Gewichtsschätzung**

- `F-MESS-07`: Ohne Waage wird das Gewicht aus Brustumfang und Körperlänge geschätzt: Gewicht (kg) ≈ Brustumfang² (cm) × Körperlänge (cm) ÷ 11.880. Brustumfang ist der Gurtumfang direkt hinter dem Widerrist, nicht der Bauchumfang. Körperlänge ist der Abstand vom Buggelenk zum Sitzbeinhöcker.
- `F-MESS-08`: Die Eingabemaske erklärt beide Maße mit einer kleinen Skizze. Das Ergebnis wird als Gewicht mit Methode „Schätzformel“ gespeichert.
- `F-MESS-09`: Geschätzte Werte erscheinen im Diagramm anders als gewogene (z. B. hohle Punkte) und tragen in Listen den Zusatz „geschätzt“.
- `F-MESS-10`: Hinweis bei der Schätzung: Bei Fohlen, Ponys und sehr schweren Rassen ist die Formel ungenauer.
- Die Formel liegt als getestete Funktion in `EquiCore`.

### 4.3 Fütterung

- `F-FUTTER-01`: Futtermittel-Stammdaten verwalten (anlegen, bearbeiten, archivieren). Sie gelten für alle Pferde.
- `F-FUTTER-02`: Futterplan pro Pferd: je Mahlzeit Futtermittel, Menge und Einheit (kg, g, Liter, Messbecher, Scheiben).
- `F-FUTTER-03`: Änderungen am Plan erzeugen eine neue Version mit Gültigkeitsdatum. So bleibt nachvollziehbar, was das Pferd an einem bestimmten Tag bekam.
- `F-FUTTER-04`: Einzelne Abweichungen oder Zusatzgaben als `FeedingLog` erfassen.
- `F-FUTTER-05`: Ansicht „Fütterung an Tag X“, die Plan und Abweichungen zusammenführt.
- `F-FUTTER-06` SOLL: Tagesmenge pro Kategorie im Zeitverlauf als Diagramm.

### 4.4 Symptome und Krankheiten

- `F-GESUND-01`: Symptom erfassen mit Kategorie, Schweregrad, Beginn, optionalem Ende, typspezifischen Details (siehe Abschnitt 3), Fotos und Notiz.
- `F-GESUND-02`: Eigene Symptom-Kategorien anlegen.
- `F-GESUND-03`: Laufende Symptome (ohne Ende) erscheinen oben in der Übersicht und lassen sich mit einem Tap beenden.
- `F-GESUND-04`: Krankheiten oder Diagnosen anlegen und Symptome sowie Behandlungen damit verknüpfen. Chronische Erkrankungen werden im Pferdeprofil dauerhaft angezeigt.
- `F-GESUND-05`: Jede Symptom-Detailansicht zeigt das Wetter am Tag des Beginns und an den drei Tagen davor (siehe 4.7).
- `F-GESUND-06`: Zeitleiste pro Pferd mit allen Symptomen, Krankheiten und Behandlungen, filterbar nach Kategorie.

### 4.5 Behandlungen und Medikamente

- `F-BEH-01`: Behandlung erfassen: Art, Datum, Dienstleister, Beschreibung, Anhänge.
- `F-BEH-02`: Arten: Tierarztbesuch, Impfung, Zahnbehandlung, Hufschmied, Osteopathie, Physiotherapie, Wurmkur, Sonstige (frei benennbar).
- `F-BEH-03`: Zu einer Behandlung beliebig viele Medikamente erfassen (Präparat, Wirkstoff, Dosis, Weg, Zeitraum, Häufigkeit).
- `F-BEH-04`: Bei Impfungen Impfstoff, Chargennummer und Krankheit(en) erfassen. Die Impfübersicht zeigt pro Krankheit die letzte Impfung und die nächste Fälligkeit.
- `F-BEH-05`: Kontakte (Tierarzt, Hufschmied usw.) einmal anlegen und wiederverwenden. Anrufen und E-Mail per Tap.
- `F-BEH-06` SOLL: Beim Speichern einer Behandlung mit passender Erinnerungsregel wird die nächste Fälligkeit automatisch neu berechnet und angezeigt.
- `F-BEH-07` KANN: Kosten erfassen und pro Jahr auswerten.

**Wartezeiten und Karenzzeiten**

- `F-BEH-08`: Pro Medikament eine Wartezeit für Lebensmittel und eine Karenzzeit für Turniere erfassen, wahlweise als Datum oder als Tage ab der letzten Gabe. Die App berechnet daraus das Enddatum. Die Werte trägt der Nutzer aus Beipackzettel oder Tierarztangabe ein. Die App liefert keine eigenen Werte.
- `F-BEH-09`: Laufende Warte- und Karenzzeiten erscheinen deutlich als Hinweis im Pferdeprofil und in der Übersicht, z. B. „Turnier-Karenzzeit bis 18. November“.
- `F-BEH-10`: Ist das Pferd laut Equidenpass Schlachttier, wird die Lebensmittel-Wartezeit zusätzlich hervorgehoben.
- `F-BEH-11` SOLL: Optionale Mitteilung, wenn eine Karenzzeit endet.
- `F-BEH-12`: Beide Enddaten sind Spalten im CSV-Export der Medikamente.

### 4.6 Erinnerungen

- `F-ERIN-01`: Erinnerungsregeln pro Pferd mit frei wählbarem Intervall (z. B. Influenza alle 6 Monate, Zahnkontrolle alle 12 Monate, Hufschmied alle 6 Wochen).
- `F-ERIN-02`: Vorlaufzeit pro Regel (z. B. 14 Tage vorher) und eine Erinnerung am Fälligkeitstag.
- `F-ERIN-03`: Vorlagen beim Anlegen anbieten (Impfung Influenza, Tetanus, Herpes, Zahn, Hufschmied, Wurmkur). Die Vorlagen enthalten nur Startwerte, die der Nutzer anpasst.
- `F-ERIN-04`: Übersicht „Demnächst fällig“ über alle Pferde, sortiert nach Datum, Überfälliges rot markiert.
- `F-ERIN-05`: Lokale Benachrichtigungen über UserNotifications. iOS erlaubt nur 64 geplante Benachrichtigungen pro App: Immer nur die nächsten fälligen planen und beim App-Start, nach jeder Änderung und per `BGAppRefreshTask` neu planen.
- `F-ERIN-06`: Aktion in der Benachrichtigung: „Erledigt“ öffnet die vorbefüllte Behandlungs-Eingabe, „In 1 Woche erinnern“ verschiebt.
- `F-ERIN-07`: Einstellung pro Gerät, ob dieses Gerät Erinnerungen anzeigt (vermeidet doppelte Hinweise auf iPhone und iPad).

### 4.7 Wetter

- `F-WETTER-01`: Wetterdaten über Bright Sky abrufen (`GET https://api.brightsky.dev/weather?lat=..&lon=..&date=..&last_date=..`). Die API liefert stündliche Werte. Die App verdichtet sie zu Tageswerten (Temperatur min/mittel/max, Niederschlagssumme, mittlere Luftfeuchte, maximale Windböe, vorherrschende Wetterlage).
- `F-WETTER-02`: Standort ist der Stall, dem das Pferd am jeweiligen Tag zugeordnet war. Ohne Stall wird kein Wetter angezeigt, sondern ein Hinweis „Stall hinterlegen“.
- `F-WETTER-03`: Ergebnisse in `WeatherDay` cachen. Vergangene Tage werden nicht erneut abgerufen, der aktuelle Tag höchstens stündlich.
- `F-WETTER-04`: Anzeige in Symptom-Details, in der Temperatur-Detailansicht und als kleine Wetterzeile in der Tagesansicht der Zeitleiste.
- `F-WETTER-05`: Quellenangabe „Wetterdaten: Deutscher Wetterdienst, über Bright Sky“ in jeder Wetteranzeige und im Impressum der App.
- `F-WETTER-06`: Offline und bei Fehlern: zwischengespeicherte Daten zeigen, sonst einen dezenten Hinweis. Kein Absturz und kein blockierender Dialog.
- `F-WETTER-07` KANN: Für Ställe außerhalb Deutschlands einen zweiten `WeatherProvider` (z. B. Open-Meteo) anbinden.

### 4.8 CSV-Export

- `F-CSV-01`: Export pro Datenbereich als eigene CSV-Datei: Pferde, Messwerte, Fütterung (Plan und Abweichungen), Symptome, Krankheiten, Behandlungen, Medikamente, Erinnerungen, Trainingseinheiten.
- `F-CSV-02`: Filter nach Pferd(en) und Zeitraum.
- `F-CSV-03`: Zwei Formate wählbar: „Excel (Deutsch)“ mit Semikolon und Dezimalkomma sowie „Standard“ mit Komma und Dezimalpunkt. Beide als UTF-8 mit BOM, Datumsangaben im ISO-8601-Format, Kopfzeile mit sprechenden Spaltennamen.
- `F-CSV-04`: Mehrere Dateien gemeinsam über `fileExporter` (mehrere Dokumente in einen gewählten Ordner) oder einzeln über das Teilen-Menü.
- `F-CSV-05`: Felder mit Trennzeichen, Anführungszeichen oder Zeilenumbrüchen werden RFC-4180-konform maskiert. Unit-Tests decken das ab.

### 4.9 Backup und Wiederherstellung

- `F-BACKUP-01`: Vollständiges Backup aller Daten inklusive Fotos und Anhängen als eine Datei mit eigenem Dateityp (z. B. `.equibackup`, Paket aus JSON plus Anhang-Dateien).
- `F-BACKUP-02`: Das JSON enthält `schemaVersion`, `appVersion` und `exportedAt`. Das Format ist in `EquiCore` definiert und dokumentiert.
- `F-BACKUP-03`: Wiederherstellen mit Vorschau (Anzahl Pferde und Einträge) und Wahl zwischen „Ersetzen“ und „Zusammenführen“. Beim Zusammenführen entscheidet die UUID, vorhandene Einträge werden nicht doppelt angelegt.
- `F-BACKUP-04`: Ein Backup einer älteren Schemaversion muss sich importieren lassen. Roundtrip-Test: Export, Import in leeren Store, Vergleich.
- `F-BACKUP-05` SOLL: Hinweis in den Einstellungen, wann das letzte Backup erstellt wurde.

### 4.10 iCloud-Synchronisierung

**Status: vorgemerkt.** Die Synchronisierung wird vorbereitet, bleibt aber über `cloudSyncEnabled = false` ausgeschaltet, bis ein Entwicklerkonto vorhanden ist (siehe Abschnitt 2). Bis dahin zeigen die Einstellungen „iCloud-Synchronisierung: noch nicht verfügbar“.

- `F-SYNC-01`: Alle Nutzerdaten außer dem Wetter-Cache werden über die private CloudKit-Datenbank zwischen iPhone und iPad synchronisiert.
- `F-SYNC-02`: Die App funktioniert vollständig ohne iCloud-Konto bzw. offline. Die Synchronisierung läuft nach, sobald möglich.
- `F-SYNC-03`: Sync-Status in den Einstellungen anzeigen (iCloud-Konto verfügbar, letzter Abgleich). Bei fehlendem Konto ein erklärender Hinweis.
- `F-SYNC-04`: Konflikte: Die zuletzt gespeicherte Änderung gewinnt (Standardverhalten von CloudKit). Das ist akzeptiert und wird nicht weiter behandelt.

### 4.11 Schnittstelle zur Trainings-App

Die Trainings-App existiert noch nicht. Diese App muss die Schnittstelle so vorbereiten, dass die Trainings-App später Einheiten aufzeichnet und diese App sie anzeigt. Apple HealthKit ist dafür ungeeignet, weil es nur Daten von Menschen kennt.

**Status: vorgemerkt.** Schema, `INTERFACE.md`, Anzeige und Deep Links werden jetzt umgesetzt. Der gemeinsame Container (`sharedTrainingContainerEnabled`) wird erst mit Entwicklerkonto aktiviert. Bis dahin liegen Trainingseinheiten im lokalen Store der App.

- `F-TRAIN-01`: Beide Apps nutzen einen gemeinsamen CloudKit-Container und eine gemeinsame App Group. Voraussetzung: Beide Apps werden unter demselben Apple-Entwicklerteam veröffentlicht.
- `F-TRAIN-02`: Datenhoheit: Diese App besitzt `Horse` (und Stammdaten), die Trainings-App besitzt `TrainingSession`. Jede App schreibt nur ihre eigenen Entitäten und liest die der anderen.
- `F-TRAIN-03`: `TrainingSession` und die gemeinsam genutzten Felder von `Horse` sind in `EquiCore` mit eigener `interfaceVersion` definiert und in `INTERFACE.md` dokumentiert.
- `F-TRAIN-04`: Bereich „Training“ pro Pferd: Liste der Einheiten, Wochenübersicht (Anzahl, Gesamtdauer, Verteilung nach Typ und Gangart) und Detailansicht mit Karte der Route.
- `F-TRAIN-05`: Deep Links: `equihealth://horse/<uuid>` öffnet ein Pferd. Ein Button „Training starten“ öffnet `equitraining://start?horse=<uuid>`, sofern die Trainings-App installiert ist, sonst wird er ausgeblendet.
- `F-TRAIN-06`: Bis die Trainings-App existiert: Ein Entwickler-Menü (nur Debug-Build) erzeugt Beispiel-Einheiten. So lassen sich Anzeige und Schnittstelle testen.
- `F-TRAIN-07` KANN: Einheiten zusätzlich manuell erfassen oder aus einer GPX-Datei importieren.

### 4.12 Daten mit anderen Personen teilen (vorgemerkt)

Diese Funktion wird **nicht** in Version 1.0 umgesetzt. Das Datenmodell muss aber später erlauben, ein Pferd mit allen Einträgen für andere Apple-Accounts freizugeben, etwa für eine Reitbeteiligung, den Stallbetreiber oder den Tierarzt.

- `F-TEIL-01` MUSS (Vorbereitung): Das Pferd ist die Wurzel aller Einträge. Jeder Eintrag hängt direkt oder über eine Behandlung am Pferd. Code darf nicht annehmen, dass Stammdaten (Futtermittel, Kontakte, Ställe) nur einem Nutzer gehören.
- `F-TEIL-02` MUSS (Vorbereitung): Optionale Felder `createdBy` und `lastModifiedBy` (Anzeigename) an allen Einträgen, vorerst ungenutzt.
- `F-TEIL-03` später: Freigabe pro Pferd über CloudKit-Freigaben (`CKShare`) mit den Rollen „Nur lesen“ und „Bearbeiten“. Bei Umsetzung prüfen, ob SwiftData das in der dann aktuellen Version direkt unterstützt, sonst für diesen Teil Core Data mit `NSPersistentCloudKitContainer`. Setzt ein Entwicklerkonto voraus.
- `F-TEIL-04` später: In der Oberfläche ist der Menüpunkt „Pferd teilen“ nur sichtbar, wenn `sharingEnabled` aktiv ist.

## 5. UI/UX und Navigation

Die App folgt den Human Interface Guidelines und sieht aus wie eine Apple-App: Systemschrift, Systemfarben, SF Symbols, Standard-Komponenten. Eigene Gestaltung beschränkt sich auf eine Akzentfarbe und Kategorie-Farben wie in Apple Health.

**Hauptnavigation** über `TabView` mit `.sidebarAdaptable`: Auf dem iPhone unten Tabs, auf dem iPad eine Seitenleiste.

| Tab | Inhalt |
| --- | --- |
| Übersicht | Wie „Zusammenfassung“ in Apple Health: laufende Symptome, demnächst fällige Termine, angeheftete Messwerte mit Mini-Diagramm, letzte Trainingseinheit. Karten per Bearbeiten-Modus an- und abheftbar. |
| Durchsuchen | Kategorien mit Farbe und SF Symbol: Messwerte, Fütterung, Symptome und Krankheiten, Behandlungen, Medikamente, Training. Jede Kategorie führt zu Listen und Detailansichten. |
| Termine | Alle Erinnerungen über alle Pferde, gruppiert nach überfällig, diesen Monat, später. |
| Suche | Volltextsuche über Notizen, Symptome, Behandlungen und Medikamente (Tab mit Such-Rolle). |

- `F-UI-01`: In der Navigationsleiste links der Pferde-Umschalter (Foto plus Name), rechts ein Profil-Button für Pferdeprofil, Pferdeverwaltung, Ställe, Kontakte, Futtermittel und Einstellungen.
- `F-UI-02`: Ein globaler „+“-Button öffnet ein Menü: Messwert, Symptom, Behandlung, Fütterung, Krankheit. Eingaben erscheinen als Sheet mit `Form`.
- `F-UI-03`: Detailansichten von Messwerten nach dem Muster von Apple Health: großer aktueller Wert, Zeitraum-Auswahl (Segmented Control), Diagramm mit Auswahl-Geste für Einzelwerte, darunter Liste.
- `F-UI-04`: iPad: Zweispaltige Layouts mit `NavigationSplitView`, wo es sinnvoll ist (Liste links, Detail rechts). Querformat und Split View/Stage Manager werden unterstützt.
- `F-UI-05`: Leere Zustände mit `ContentUnavailableView` und einer klaren ersten Aktion (z. B. „Erstes Pferd anlegen“).
- `F-UI-06`: Onboarding beim ersten Start in höchstens drei Schritten: Pferd anlegen, Stall setzen (optional), Benachrichtigungen erlauben (optional).
- `F-UI-07`: Dunkelmodus, Dynamic Type bis zu den größten Barrierefreiheitsgrößen, VoiceOver-Beschriftungen für alle Bedienelemente und Diagramme (Audio Graphs über Swift Charts).
- `F-UI-08` SOLL: Widgets für den Home-Bildschirm (nächster Termin, letzter Gewichtswert). Vorgemerkt: Widgets brauchen für den Datenzugriff eine App Group und damit ein Entwicklerkonto.
- `F-UI-09` KANN: App Intents für Kurzbefehle und Siri, z. B. „Temperatur für Luna eintragen“.

## 6. Nicht-funktionale Anforderungen

Datenschutz und Zuverlässigkeit haben Vorrang vor Funktionsumfang: Die Daten verlassen das Gerät nur in Richtung iCloud des Nutzers und als Wetterabfrage.

| ID | Bereich | Anforderung |
| --- | --- | --- |
| `NF-01` | Datenschutz | Kein eigener Server, kein Tracking, keine Analyse-SDKs, kein Login. Wetterabfragen senden nur auf 2 Nachkommastellen gerundete Koordinaten (ca. 1 km). |
| `NF-02` | Datenschutz | Privacy Manifest (`PrivacyInfo.xcprivacy`) und App-Store-Datenschutzangaben „Keine Daten erfasst“, sofern zutreffend. |
| `NF-03` | Offline | Alle Funktionen außer Wetterabruf und Synchronisierung arbeiten ohne Netz. |
| `NF-04` | Leistung | Flüssiges Scrollen und Diagramme ohne Ruckeln mit Testdaten von 10 Pferden über 10 Jahre (ca. 50.000 Einträge). Ein Generator für diese Testdaten liegt im Debug-Menü. |
| `NF-05` | Leistung | Datenbankzugriffe für Export, Import und Auswertungen laufen nicht auf dem Main Actor. |
| `NF-06` | Zuverlässigkeit | Kein Datenverlust bei App-Abbruch während der Eingabe oder des Imports. Import läuft transaktional: ganz oder gar nicht. |
| `NF-07` | Barrierefreiheit | VoiceOver, Dynamic Type, ausreichender Kontrast, keine Information nur über Farbe. |
| `NF-08` | Lokalisierung | Alle Texte in String Catalogs, Zahlen-, Datums- und Einheitenformatierung über `FormatStyle` passend zur Region. |
| `NF-09` | Codequalität | Formatierung mit `swift-format` (Teil der Swift-Toolchain). Keine Compiler-Warnungen. |
| `NF-10` | Tests | Unit-Tests für Fälligkeitsberechnung, Wetter-Verdichtung, CSV-Maskierung, Backup-Roundtrip und Migrationen. UI-Tests für: Pferd anlegen, Messwert eintragen, Symptom mit Wetter ansehen, Export. |
| `NF-11` | Haftung | Hinweis in Onboarding und Einstellungen: Die App ersetzt keine tierärztliche Beratung. Referenzbereiche sind Nutzereinstellungen. |

**Vertrieb:** Geplant ist ein Einmalkauf, die App wird aber vorerst nicht veröffentlicht. Version 1.0 enthält deshalb kein StoreKit, keine Paywall und keine Lizenzprüfung. Ob die App ein Bezahl-Download wird oder kostenlos mit einmaliger Freischaltung, wird vor der Veröffentlichung entschieden. Es gibt keine Abo-Logik.

## 7. Umsetzungsphasen und Akzeptanzkriterien

Zehn Phasen in fester Reihenfolge. Jede Phase ist abgeschlossen, wenn das Projekt ohne Warnungen für iPhone und iPad baut, alle Tests grün sind und die genannten Kriterien erfüllt sind.

0. **Projektgerüst**: Xcode-Projekt mit der neuesten stabilen Swift-Version, Swift Package `EquiCore`, SwiftData-Schema V1 (CloudKit-konform), `FeatureFlags`, vorbereitete, nicht eingebundene Entitlements, `CLAUDE.md`, `SPEC.md`, `OPEN_QUESTIONS.md`.
   - Abnahme: App startet auf iPhone- und iPad-Simulator und auf einem echten Gerät mit kostenlosem Apple-Account (Personal Team) mit leerer Tab-Struktur. Ein Test prüft, dass alle Modelle die CloudKit-Regeln einhalten.
1. **Pferde und Messwerte** (4.1, 4.2, Grundgerüst von 5).
   - Abnahme: Zwei Pferde anlegen, umschalten, für jedes Gewicht und Temperatur eintragen. Diagramm mit Zeiträumen und Referenzband funktioniert.
2. **Gesundheit** (4.4, 4.5): Symptome, Krankheiten, Behandlungen, Medikamente, Kontakte, Zeitleiste.
   - Abnahme: Kolik mit Details und Foto erfassen, mit Tierarztbesuch und Medikament verknüpfen, alles erscheint in der Zeitleiste.
3. **Fütterung** (4.3).
   - Abnahme: Plan anlegen, ändern, Ansicht „Fütterung an Tag X“ zeigt für einen Tag vor der Änderung den alten Plan.
4. **Erinnerungen** (4.6).
   - Abnahme: Regel „Influenza alle 6 Monate, 14 Tage Vorlauf“ ergibt nach Erfassen einer Impfung das richtige Fälligkeitsdatum. Benachrichtigung erscheint, Aktion „Erledigt“ öffnet die Eingabe.
5. **Wetter** (4.7).
   - Abnahme: Für einen Stall in Deutschland zeigt ein Symptom vom letzten Jahr Tageswerte. Zweiter Aufruf lädt aus dem Cache (im Test mit Mock-Provider prüfbar). Ohne Netz kein Fehlerdialog.
6. **Export und Backup** (4.8, 4.9).
   - Abnahme: CSV in beiden Formaten öffnet sich in Excel bzw. Numbers mit korrekten Umlauten und Spalten. Backup-Roundtrip-Test ist grün.
7. **iCloud-Sync und Teilen vorbereiten** (4.10, 4.12): Sync-Code hinter `cloudSyncEnabled`, Vorbereitungsfelder für das Teilen, `ENABLE_CLOUDKIT.md` mit Schritt-für-Schritt-Anleitung für den Tag, an dem ein Entwicklerkonto vorhanden ist.
   - Abnahme: Mit allen Flags auf `false` baut und startet die App ohne iCloud-Entitlement fehlerfrei. Die Einstellungen zeigen „noch nicht verfügbar“. Die Anleitung ist vollständig.
8. **Trainings-Schnittstelle** (4.11).
   - Abnahme: Beispiel-Einheiten aus dem Debug-Menü erscheinen pro Pferd mit Wochenübersicht und Karte (lokaler Store). `INTERFACE.md` beschreibt Schema, Deep Links und den späteren gemeinsamen Container vollständig.
9. **Feinschliff**: Barrierefreiheits-Durchgang, Englisch-Lokalisierung, App-Icon-Platzhalter. Widgets und App-Store-Vorbereitung bleiben vorgemerkt.
   - Abnahme: Accessibility Inspector meldet keine Probleme in den Hauptansichten.

**Startprompt für Claude Code** (zum Kopieren):

```
Lies SPEC.md vollständig. Nutze die neueste stabile Swift-Version.
Die App muss ohne bezahltes Apple-Entwicklerkonto laufen (Abschnitt 2, "Betrieb ohne Entwicklerkonto").
Setze Phase 0 um und halte dich an die CloudKit-Regeln aus Abschnitt 2.
Arbeite danach Phase für Phase. Am Ende jeder Phase: Build für iPhone und iPad, alle Tests ausführen,
Abnahmekriterien prüfen, kurz zusammenfassen und auf meine Freigabe warten.
Vorgemerkte Funktionen (iCloud-Sync, gemeinsamer Trainings-Container, Teilen, Widgets) nur vorbereiten, nicht aktivieren.
Annahmen dokumentierst du in OPEN_QUESTIONS.md.
```

## 8. Entscheidungen und offene Fragen

Alle zehn Punkte sind entschieden. Neue Fragen, die während der Umsetzung entstehen, sammelt Claude Code in `OPEN_QUESTIONS.md`.

| Thema | Entscheidung (9. November 2026) |
| --- | --- |
| Swift-Version | Immer die neueste stabile Version |
| Mindestversion | iOS/iPadOS 27 |
| Mac | Nur „Designed for iPad“ auf Apple-Silicon-Macs, keine eigene Mac-Version |
| Entwicklerkonto | Derzeit keins. Die App läuft ohne Konto. iCloud-Sync, gemeinsamer Trainings-Container, Teilen und Widgets sind vorgemerkt. |
| Apple Watch | Diese App bekommt keine Watch-Komplikation |
| Daten teilen | Eingeplant (4.12), aber nicht implementiert |
| Gewichtsschätzung | Ja, aus Brustumfang und Körperlänge (4.2) |
| Warte- und Karenzzeiten | Ja, hervorgehoben (4.5) |
| Geschäftsmodell | Einmalkauf, kein Abo |
| Veröffentlichung | Vorerst nicht. Kein StoreKit in Version 1.0. |

## Quellen

- [Bright Sky: JSON-API für DWD-Daten, kostenlos und ohne API-Schlüssel](https://brightsky.dev/)
- [Bright Sky Quellcode und Dokumentation (GitHub)](https://github.com/jdemaeyer/brightsky)
- [What’s new in SwiftData, WWDC26](https://developer.apple.com/videos/play/wwdc2026/274/)

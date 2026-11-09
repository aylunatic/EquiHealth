# EquiHealth – Hinweise für Claude Code

Native iPhone-/iPad-App (SwiftUI, SwiftData) zum Erfassen von Gesundheitsdaten von Pferden – ähnlich Apple Health.

## Quellen der Wahrheit

- `SPEC.md` – verbindliche Anforderungen. Bei Widerspruch gilt SPEC.md, nicht diese Datei.
- `PROGRESS.md` – aktueller Stand. **Zu Beginn jeder Sitzung zuerst lesen.**
- `OPEN_QUESTIONS.md` – getroffene Annahmen und offene Fragen.
- `INTERFACE.md` – Schnittstelle zur späteren Trainings-App (ab Phase 8).

## Arbeitsweise

- Arbeite die Phasen aus SPEC.md, Abschnitt 7, der Reihe nach ab. Immer nur eine Phase pro Durchgang.
- Am Ende jeder Phase:
  1. Build für iPhone **und** iPad ohne Warnungen.
  2. Alle Tests ausführen, alle grün.
  3. Abnahmekriterien der Phase prüfen.
  4. `PROGRESS.md` aktualisieren: erledigt, nächster Schritt, bekannte Probleme.
  5. Git-Commit mit Phase und Anforderungs-IDs, z. B. `Phase 2: Symptome erfassen (F-GESUND-01, F-GESUND-03)`.
  6. Kurz zusammenfassen und auf meine Freigabe warten.
- Auch innerhalb einer Phase nach jedem sinnvollen Zwischenschritt committen, damit bei einer Unterbrechung nichts verloren geht.
- Bei Unklarheiten nicht anhalten: sinnvolle Annahme treffen, im Code mit `// ANNAHME:` markieren und in `OPEN_QUESTIONS.md` eintragen.
- Anforderungs-IDs (z. B. `F-MESS-07`) in Commits, Testnamen und bei Bedarf in Kommentaren nennen.

## Harte Regeln

- **Kein bezahltes Entwicklerkonto vorhanden.** Die App muss im Simulator und mit kostenlosem Apple-Account (Personal Team) auf einem echten Gerät laufen.
  - Keine iCloud-, CloudKit-, App-Group- oder Push-Capabilities in einem Target aktivieren.
  - Alle Feature-Flags in `EquiCore/FeatureFlags` bleiben `false` (`cloudSyncEnabled`, `sharedTrainingContainerEnabled`, `sharingEnabled`, `widgetsEnabled`).
  - SwiftData läuft lokal (`cloudKitDatabase: .none`).
- **Datenmodell trotzdem CloudKit-konform:** alle Eigenschaften optional oder mit Standardwert, keine `@Attribute(.unique)`, alle Beziehungen optional mit Inverse, große Binärdaten mit `.externalStorage`. Nach Phase 0 nur additive Schemaänderungen über `VersionedSchema`.
- Keine Fremdbibliotheken. Nur Apple-Frameworks.
- Kein HealthKit (unterstützt keine Tiere), kein StoreKit, keine Analyse- oder Tracking-SDKs.
- Einziger Netzwerkzugriff: Bright Sky (`https://api.brightsky.dev`), Koordinaten auf 2 Nachkommastellen gerundet.
- Keine medizinischen Empfehlungen oder Grenzwerte erfinden. Referenzbereiche und Wartezeiten sind Nutzereingaben.

## Technik und Konventionen

- Neueste stabile Swift-Version, Swift-6-Sprachmodus, Strict Concurrency. Mindestversion iOS/iPadOS 27.
- Datenmodell, Export-/Backup-Formate, Schätzformeln und Schnittstellen-Typen liegen im lokalen Swift Package `EquiCore`.
- Einheiten intern in SI (kg, cm, °C), Anzeige über `Measurement` und `FormatStyle`.
- Netzwerk- und Wetterlogik hinter dem Protokoll `WeatherProvider`; in Tests ein Mock.
- Datenbankarbeit für Export, Import und Auswertungen nicht auf dem Main Actor.
- UI: nur Systemkomponenten, SF Symbols, Dynamic Type, VoiceOver-Beschriftungen. Keine festen Farben außer Akzent- und Kategoriefarben aus dem Asset-Katalog.
- Sprache: Code, Bezeichner und Kommentare auf Englisch. Alle UI-Texte auf Deutsch über String Catalogs (Englisch vorbereitet). Antworten an mich auf Deutsch.
- Formatierung mit `swift format` aus der Toolchain.

## Befehle

In Phase 0 mit den tatsächlichen Scheme- und Simulator-Namen ergänzen (`xcrun simctl list devices available`):

```bash
# Build iPhone
xcodebuild -scheme EquiHealth -destination 'platform=iOS Simulator,name=<iPhone-Simulator>' build

# Build iPad
xcodebuild -scheme EquiHealth -destination 'platform=iOS Simulator,name=<iPad-Simulator>' build

# Tests (App und EquiCore)
xcodebuild -scheme EquiHealth -destination 'platform=iOS Simulator,name=<iPhone-Simulator>' test
swift test --package-path EquiCore

# Formatierung prüfen
swift format lint -r EquiHealth EquiCore
```

## Projektstruktur (Ziel)

```
EquiHealth/            App-Target
  Features/            je Bereich ein Ordner (Horses, Measurements, Feeding, Health,
                       Treatments, Reminders, Weather, Export, Settings, Training)
  Shared/              gemeinsame Views und Hilfen
  Resources/           Asset-Katalog, String Catalogs
EquiCore/              Swift Package: Modelle, Formate, Schnittstelle, FeatureFlags
EquiHealthTests/       Unit-Tests (Swift Testing)
EquiHealthUITests/     UI-Tests
Entitlements/          vorbereitete, NICHT eingebundene iCloud-/App-Group-Entitlements
```

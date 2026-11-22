# CKW Dynamische Strompreise für Home Assistant

Dieses öffentliche Teilprojekt ruft dynamische CKW-Tarifdaten ab und zeigt Beispiele für Sensoren, Preisfenster, Empfehlungen und Dashboard-Karten. Es ist ein unabhängiges Community-Beispiel und wird nicht offiziell von CKW oder Home Assistant unterstützt.

![CKW-Dashboard mit Ladezeitfenstern, Preisempfehlungen und dynamischen Strompreisen](docs/assets/dashboard-preview.png)

Die Dashboard-Vorschau zeigt Beispielwerte vom 22. November 2026. Preise und Zeitfenster ändern sich laufend.

## Inhalt

- `packages/ckw_tarife.yaml`: Home Assistant Package mit der CKW-REST-Datenquelle
- `docs/anleitung.md`: Einrichtung und Code der Sensoren und Helfer, Empfehlungen, Benachrichtigungen und automatische Aktualisierung
- `docs/auto-laden.md`: günstigstes und teuerstes nicht überlappendes 5-Stunden-Fenster fürs Elektroauto
- `docs/dashboard.md`: Dashboard-Karten und Anordnung
- `docs/examples/rest-sensor.yaml`: eigenständiges REST-Beispiel
- `docs/assets/dashboard-preview.png`: Dashboard-Vorschau
- `docs/CKW_Dynamischer_Tarif_Home_Assistant_Anleitung.docx`: Anleitung als Word-Dokument
- `docs/source/`: archivierte ursprüngliche Projektübergabe samt Anhängen und Screenshots

## Installation

Kopiere `packages/ckw_tarife.yaml` in den Ordner `packages/` deiner Home-Assistant-Konfiguration. Stelle sicher, dass deine `configuration.yaml` `packages/` mit `!include_dir_named packages` einbindet. Die übrigen Sensoren und Karten sind in der [Anleitung](docs/anleitung.md) beschrieben.

Prüfe den Tarifnamen und die tatsächlich erzeugten Entity-IDs deiner Installation. CKW veröffentlicht Preise für den Folgetag erst später am Tag; Beispielwerte in Screenshots sind Momentaufnahmen.

## Hinweise zur Veröffentlichung

Die aktuelle und massgebliche Anleitung liegt unter `docs/anleitung.md`; die Dashboard-Referenz steht unter `docs/dashboard.md`. Die Dateien in `docs/source/` sind historische Arbeitsunterlagen und keine separate, aktuelle Anleitung. Änderungen gehören in die aktiven Dateien ausserhalb des Archivs.

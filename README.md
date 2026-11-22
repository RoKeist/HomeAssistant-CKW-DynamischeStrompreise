# CKW Dynamische Strompreise für Home Assistant

Dieses öffentliche Teilprojekt ruft dynamische CKW-Tarifdaten ab und zeigt Beispiele für Sensoren, Preisfenster, Empfehlungen und Dashboard-Karten. Es ist ein unabhängiges Community-Beispiel und wird nicht offiziell von CKW oder Home Assistant unterstützt.

![CKW-Dashboard mit Ladezeitfenstern, Preisempfehlungen und dynamischen Strompreisen](docs/assets/dashboard-preview.png)

Die Dashboard-Vorschau zeigt Beispielwerte vom 22. November 2026. Preise und Zeitfenster ändern sich laufend.

## Inhaltsverzeichnis

- [CKW-Tarifdaten-Package](packages/ckw_tarife.yaml)
- [1. CKW-Package einbinden](docs/anleitung.md#1-ckw-package-einbinden)
- [Auto-Laden-Zeitfenster](docs/auto-laden.md)
- [CKW-Dashboard für Home Assistant](docs/dashboard.md)

Prüfe den Tarifnamen und die tatsächlich erzeugten Entity-IDs deiner Installation. CKW veröffentlicht Preise für den Folgetag erst später am Tag; Beispielwerte in Screenshots sind Momentaufnahmen.

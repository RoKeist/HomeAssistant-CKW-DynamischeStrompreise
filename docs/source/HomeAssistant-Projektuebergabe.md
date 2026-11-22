# HomeAssistant – Übergabe aus dem bisherigen Chat

## Ziel

Dieses Projekt dient der Weiterentwicklung von Home Assistant: Konfigurationen, Templates, Sensoren, Automationen und Dashboards. Der bisherige Arbeitsstrang betrifft dynamische CKW-Strompreise.

## CKW-Datenquelle

- REST-Daten-Sensor: `sensor.ckw_tarifdaten`
- Preiswerte: Attribut `prices`, je Viertelstunde; `integrated[0].value` enthält den Preis in CHF/kWh
- Regelmässiger Abruf bislang alle 6 Stunden
- Manuelle Aktualisierung über Aktion `homeassistant.update_entity` mit Ziel `sensor.ckw_tarifdaten`
- Morgenpreise stehen erst nach Veröffentlichung durch CKW zur Verfügung (im Chat typischerweise um die Mittagszeit beobachtet).

Das genaue REST-Beispiel liegt in `configuration.yaml.example` und in den beigefügten Anleitungen.

## Bisherige Helfer und Sensoren

Aktueller Preis: `sensor.ckw_aktueller_preis`

Heute:
- `sensor.ckw_guenstigstes_zeitfenster_von_2h`
- `sensor.ckw_zweitguenstigstes_zeitfenster_von_2h`
- `sensor.ckw_teuerstes_zeitfenster_von_2h`
- `sensor.ckw_zweitteuerstes_zeitfenster_von_2h`

Morgen:
- `sensor.ckw_guenstigstes_zeitfenster_morgen_von_2h`
- `sensor.ckw_zweitguenstigstes_zeitfenster_morgen_von_2h`
- `sensor.ckw_teuerstes_zeitfenster_morgen_von_2h`
- `sensor.ckw_zweitteuerstes_zeitfenster_morgen_von_2h`

Empfehlungen: `sensor.ckw_preisempfehlung` und `sensor.ckw_preisempfehlung_morgen`. Die Morgenauswertung soll nur Daten des morgigen Datums verarbeiten. Ein 2-Stunden-Fenster besteht aus acht Viertelstundenwerten; die beiden empfohlenen Fenster einer Kategorie sollen sich nicht überlappen und chronologisch ausgegeben werden.

## Dashboard-Stand

Das Dashboard umfasst den aktuellen Preis, die vier günstigsten/teuersten 2-Stunden-Fenster für heute und morgen, zwei farbige Preisempfehlungskarten, Preisverlaufkarten, Preislisten für heute und morgen sowie eine Schaltfläche zum manuellen Abruf der CKW-Daten. Die Hinweise in den Empfehlungen sind grün für stromintensive Arbeiten und rot für Verbrauch reduzieren. Zeiten sind chronologisch zu sortieren; am Ende der Hinweistexte stehen keine Punkte.

## Automationen

Es wurden E-Mail-Benachrichtigungen für den Beginn des günstigsten, zweitgünstigsten, teuersten und zweitteuersten Fensters geplant. Diese Benachrichtigungen sind als Test gedacht, um zu prüfen, ob Aktionen oder Sensoren tatsächlich zu den ermittelten Zeitpunkten schalten. Eine E-Mail allein schaltet kein Gerät.

## Fehlerbilder und Erkenntnisse

- Ein Template, das im Entwicklerwerkzeug funktioniert, kann als Sensor `unbekannt` anzeigen, wenn es nicht im Zustands-Template eines Template-Sensor-Helfers angelegt wurde.
- Ein Dashboard-Template gab `interval is undefined` aus, weil eine verwendete Variable nicht definiert war.
- Die Morgen-Preisempfehlung zeigte zunächst keine Daten, solange die vier Sensoren für morgen keine Werte hatten. Später zeigte sie Text; sie soll analog zur heutigen Empfehlung formatiert werden.
- Ein leerer Sensorzustand oder `unknown` sollte durch Prüfung der Quell-Sensorattribute und des Helfer-Templates eingegrenzt werden.

## Dateien in diesem Paket

- `README.md`: Anleitung und Übersicht der Helfer/Sensoren
- `dashboard.md`: Erklärung und YAML-Beispiele für das Dashboard
- `configuration.yaml.example`: REST-Datenquelle und Beispielkonfiguration
- `dashboard-preview.png`: Dashboard-Momentaufnahme
- `CKW_Dynamischer_Tarif_Home_Assistant_Anleitung.docx`: Anleitung als Word-Dokument
- `chat-anhange-und-screenshots.zip`: die im bisherigen Arbeitsverzeichnis vorhandenen Anhänge und Screenshots

Die Screenshots enthalten Momentaufnahmen und Beispielwerte. Zeitangaben und Preise sind nicht dauerhaft gültig.

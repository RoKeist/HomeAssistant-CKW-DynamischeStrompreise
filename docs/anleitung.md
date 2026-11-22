# CKW dynamischer Tarif für Home Assistant

Community-Beispiele, um die öffentliche CKW-API für dynamische Tarife in Home Assistant abzufragen, die Preise für heute und morgen anzuzeigen und den aktuellen Preis in Automationen zu verwenden.

Dieses unabhängige Community-Projekt steht in keiner Verbindung zu CKW oder zum Home-Assistant-Projekt und wird von diesen weder unterstützt noch offiziell bestätigt.

## Inhaltsverzeichnis

- [Voraussetzungen](#voraussetzungen)
- [1. CKW-Tarifdaten-Package einbinden](#1-ckw-tarifdaten-package-einbinden)
- [2. Template-Sensor für den aktuellen Preis](#2-template-sensor-für-den-aktuellen-preis)
- [Hoch- & Tiefpreis Zeitfenster](zeitfenster-2h.md)
- [Preisempfehlungen für heute und morgen](preisempfehlungen.md)
- [3. Automationsbeispiel](#3-automationsbeispiel)
- [Preise nach Mittag automatisch aktualisieren](#preise-nach-mittag-automatisch-aktualisieren)
- [Fehlerbehebung](#fehlerbehebung)
- [Quellen](#quellen) · [Mitwirken](#mitwirken)

**Weitere Anleitungen:** [Karten im Dashboard anordnen](dashboard.md#karten-im-dashboard-anordnen) · [Auto-Laden-Zeitfenster](auto-laden.md)

CKW veröffentlicht Viertelstundenwerte. Ein vollständiger Tag umfasst 96 Werte. Das Tarifprofil für den Folgetag ist ungefähr ab 12 Uhr mittags verfügbar. Der API-Wert `integrated` wird in CHF/kWh angegeben und entspricht nicht zwingend dem vollständigen Rechnungsbetrag: Steuern, Messentgelte, Konzessionsabgaben und Leistungspreise können darin fehlen.

## Voraussetzungen

- Ein CKW-Tarif und ein Tarifname, die zu deinem Vertrag passen. `home_dynamic` gilt bei einem Jahresverbrauch unter 50 MWh; `business_dynamic` bei einem Jahresverbrauch über 50 MWh.
- Ein Datei-Editor oder eine andere Möglichkeit, `configuration.yaml` zu bearbeiten.

Das CKW-Package erstellt `sensor.ckw_tarifdaten`. Die übrigen Vorlagen und Karten verwenden diese Entität als Datenquelle.

## 1. CKW-Tarifdaten-Package einbinden

Das einzige Konfigurationsbeispiel für den CKW-Datenabruf ist das [CKW-Tarifdaten-Package](../packages/ckw_tarife.yaml). Es fragt alle sechs Stunden den Tarif `home_dynamic` mit dem Datentyp `integrated` für heute und morgen ab. Die Abfrage verwendet die Zeitzone von Home Assistant.

Kopiere die Datei in den Ordner `packages/` deiner Home-Assistant-Konfiguration. Stelle sicher, dass `configuration.yaml` diesen Ordner einbindet:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Wenn `homeassistant:` in deiner `configuration.yaml` bereits vorhanden ist, ergänze dort nur den Eintrag `packages:`.

Das Package erstellt `sensor.ckw_tarifdaten` mit dem Zustand `OK`. Die veröffentlichten Preise und der Veröffentlichungszeitpunkt stehen in den Attributen `prices` und `publication_timestamp`. Prüfe nach dem Einbinden, dass die Entität und beide Attribute vorhanden sind.

## 2. Template-Sensor für den aktuellen Preis

Erstelle einen Template-Sensor-Helfer unter **Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Template → Sensor**. Verwende als Namen `CKW aktueller Preis` und füge dieses Template ins Zustandsfeld ein:

```jinja
{% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set now_ts = as_timestamp(now()) %}
{% set current = namespace(value=none) %}
{% for p in prices %}
  {% set start = as_timestamp(p.start_timestamp, 0) %}
  {% set end = as_timestamp(p.end_timestamp, 0) %}
  {% if start <= now_ts < end and p.integrated is defined and p.integrated | count > 0 %}
    {% set current.value = p.integrated[0].value %}
  {% endif %}
{% endfor %}
{{ current.value }}
```

Setze die Einheit auf `CHF/kWh`. Lasse die Geräteklasse leer. Falls der Helfer eine Zustandsklasse anbietet, wähle `measurement` (Messwert). Speichere den Helfer und prüfe die von Home Assistant erstellte Entitäts-ID. Die folgenden Beispiele verwenden `sensor.ckw_aktueller_preis`; ersetze sie durch die tatsächlich erstellte ID, falls sie abweicht.

Die Einrichtung der acht 2-Stunden-Zeitfenster ist in der separaten [Anleitung zu Hoch- & Tiefpreis Zeitfenstern](zeitfenster-2h.md) beschrieben. Die beiden Sensor-Helfer für die [Preisempfehlungen](preisempfehlungen.md) sind ebenfalls ausgelagert.
## 3. Automationsbeispiel

Dieses Beispiel wird ausgelöst, wenn der aktuelle Preis die Schwelle von 0,20 CHF/kWh (20 Rp./kWh) nach unten überschreitet. Ergänze unter `actions:` die gewünschte Aktion.

```yaml
alias: CKW-Preis unter 20 Rp. pro kWh
triggers:
  - trigger: numeric_state
    entity_id: sensor.ckw_aktueller_preis
    below: 0.20
actions:
  # Ergänze hier die gewünschte Aktion.
  - action: persistent_notification.create
    data:
      title: CKW-Preis ist tief
      message: Der aktuelle CKW-Preis liegt unter 20 Rp./kWh.
mode: single
```

Ein numerischer Zustandsauslöser reagiert, wenn der Sensor die Schwelle überschreitet. Soll eine Aktion in jedem passenden 15-Minuten-Intervall ausgeführt werden, verwende stattdessen einen periodischen Zeitauslöser mit einer separaten Preisbedingung.

### E-Mail bei Beginn eines teuren Zeitfensters

Diese Automation prüft den Startzeitpunkt der beiden teuersten 2-Stunden-Zeitfenster. Wenn eines beginnt, sendet Home Assistant eine Benachrichtigung. Damit daraus eine E-Mail wird, muss in Home Assistant bereits ein E-Mail-Benachrichtigungsdienst eingerichtet sein. Ersetze `notify.deine_e_mail_benachrichtigung` durch die Entity-ID deines Benachrichtigungsdienstes.

Die Vorlagen zerlegen den Sensorzustand im Format `Datum · Start–Ende · Preis` und lösen nur dann aus, wenn Datum und Startzeit mit der aktuellen Minute übereinstimmen. Verwende die tatsächlichen Entity-IDs deiner Helfer, falls Home Assistant sie anders angelegt hat.

```yaml
alias: CKW – E-Mail bei teurem Zeitfenster
description: >-
  Sendet eine E-Mail bei Beginn des teuersten oder zweitteuersten
  2-Stunden-Fensters.
triggers:
  - trigger: template
    id: teuerstes
    value_template: >-
      {% set s = states('sensor.ckw_teuerstes_zeitfenster_von_2h') %}
      {% set teile = s.split(' · ') %}
      {% if teile | count >= 2 %}
        {% set startzeit = teile[1].split('–')[0] %}
        {{ now().strftime('%Y-%m-%d %H:%M') == teile[0] ~ ' ' ~ startzeit }}
      {% else %}
        false
      {% endif %}
  - trigger: template
    id: zweitteuerstes
    value_template: >-
      {% set s = states('sensor.ckw_zweitteuerstes_zeitfenster_von_2h') %}
      {% set teile = s.split(' · ') %}
      {% if teile | count >= 2 %}
        {% set startzeit = teile[1].split('–')[0] %}
        {{ now().strftime('%Y-%m-%d %H:%M') == teile[0] ~ ' ' ~ startzeit }}
      {% else %}
        false
      {% endif %}
conditions: []
actions:
  - action: notify.send_message
    target:
      entity_id: notify.deine_e_mail_benachrichtigung
    data:
      title: >-
        {% if trigger.id == 'teuerstes' %}
          Teuerstes CKW-Zeitfenster beginnt
        {% else %}
          Zweitteuerstes CKW-Zeitfenster beginnt
        {% endif %}
      message: >-
        {% if trigger.id == 'teuerstes' %}
          Das teuerste 2-Stunden-Zeitfenster beginnt jetzt:
          {{ states('sensor.ckw_teuerstes_zeitfenster_von_2h') }}
        {% else %}
          Das zweitteuerste 2-Stunden-Zeitfenster beginnt jetzt:
          {{ states('sensor.ckw_zweitteuerstes_zeitfenster_von_2h') }}
        {% endif %}
mode: single
```

Füge die Automation im YAML-Editor ein. Wenn du sie direkt als Eintrag in `automations.yaml` speicherst, setze einen Bindestrich vor `alias` und rücke die restlichen Zeilen um zwei Leerzeichen ein. Der E-Mail-Dienst muss in Home Assistant eingerichtet und getestet sein.
## Preise nach Mittag automatisch aktualisieren

Die CKW-Preise für den Folgetag werden üblicherweise ungefähr ab 12 Uhr veröffentlicht. Ein manueller Dashboard-Button kann `sensor.ckw_tarifdaten` sofort aktualisieren; seine Konfiguration steht unter [CKW-Preise aktualisieren](dashboard.md#ckw-preise-aktualisieren). Die folgende Automation startet täglich um 12:05 Uhr, aktualisiert den CKW-Sensor und prüft danach, ob Preise für morgen vorhanden sind. Falls nicht, versucht sie es bis zu acht weitere Male im Abstand von 15 Minuten. Sobald mindestens ein Preis für morgen eintrifft, endet die Wiederholung.

Füge den Eintrag in `automations.yaml` ein. Die Datei wird in diesem Beispiel über `automation: !include automations.yaml` in `configuration.yaml` eingebunden. Ersetze `sensor.ckw_tarifdaten`, falls deine Installation eine andere Entity-ID verwendet.

```yaml
- alias: CKW-Preise nach Mittag aktualisieren
  description: Aktualisiert die Preise und wiederholt den Abruf, bis CKW Preise für morgen liefert.
  triggers:
    - trigger: time
      at: "12:05:00"
  actions:
    - action: homeassistant.update_entity
      target:
        entity_id: sensor.ckw_tarifdaten

    - delay: "00:02:00"

    - repeat:
        while:
          - condition: template
            value_template: >-
              {% set morgen = (today_at('00:00') + timedelta(days=1)).strftime('%Y-%m-%d') %}
              {% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
              {% set ns = namespace(found=false) %}
              {% for p in prices %}
                {% if p.start_timestamp is defined and p.start_timestamp[:10] == morgen %}
                  {% set ns.found = true %}
                {% endif %}
              {% endfor %}
              {{ not ns.found and repeat.index <= 8 }}
        sequence:
          - delay: "00:15:00"
          - action: homeassistant.update_entity
            target:
              entity_id: sensor.ckw_tarifdaten
          - delay: "00:02:00"

  mode: single
```

Der führende Bindestrich vor `alias` ist erforderlich, wenn du den Code als neuen Listeneintrag in `automations.yaml` einfügst. Wenn du im YAML-Editor eine einzelne Automation bearbeitest, entferne diesen ersten Bindestrich. Eine von Home Assistant verwaltete Automation, die nicht in `automations.yaml` liegt, lässt sich gegebenenfalls nur im grafischen Editor ändern.

Die Wiederholung ist auf acht zusätzliche Abrufe begrenzt. Die manuelle Schaltfläche bleibt als Möglichkeit für einen sofortigen Abruf bestehen. Die Aktion `homeassistant.update_entity` löst eine Aktualisierung ausserhalb des normalen `scan_interval` aus.

## Fehlerbehebung

- **Der CKW-Sensor zeigt `OK`:** Das ist erwartungsgemäss. Die Preise sind Attribute des Sensors, nicht sein Zustand.
- **Die Entität erscheint nicht auf der Zuständeseite:** Verwende in Home Assistant den Entitätsfilter und nicht die Seitensuche des Browsers. Suche nach dem Anzeigenamen des Sensors oder nach `ckw`.
- **Die Karte für morgen ist leer:** Prüfe den letzten `start_timestamp` im Attribut `prices` sowie `publication_timestamp`. CKW veröffentlicht die Preise für den Folgetag ungefähr um die Mittagszeit.
- **Es erscheinen nur die heutigen Preise:** Prüfe im CKW-Package, ob die Abfrage `start_timestamp` und `end_timestamp` für heute und morgen enthält. Aktualisiere den Sensor erneut, nachdem CKW das nächste Tagesprofil veröffentlicht hat.
- **Dashboard-Karte oder Anordnung stimmt nicht:** Siehe [Karten im Dashboard anordnen](dashboard.md#karten-im-dashboard-anordnen).
- **Die Automation wird nicht ausgelöst:** Prüfe die tatsächliche Entitäts-ID des aktuellen Preis-Sensors und ob sein Zustand numerisch ist, zum Beispiel `0.1583`.

## Quellen

- [CKW: Technische Informationen zum dynamischen Tarif und zur API](https://www.ckw.ch/energie/strom/dynamischer-tarif-fuer-privatkunden-und-unternehmen/dynamischer-tarif-technische-voraussetzungen)

## Mitwirken

Fehlermeldungen und Pull Requests mit Korrekturen, getesteten Verbesserungen oder Hinweisen zur Kompatibilität sind willkommen. Entferne persönliche Angaben, Tokens, IP-Adressen und private Home-Assistant-Entitätsnamen, bevor du Konfigurationen oder Protokolle teilst.

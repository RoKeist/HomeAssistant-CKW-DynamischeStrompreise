# CKW dynamischer Tarif für Home Assistant

Community-Beispiele, um die öffentliche CKW-API für dynamische Tarife in Home Assistant abzufragen, die Preise für heute und morgen anzuzeigen und den aktuellen Preis in Automationen zu verwenden.

Dieses unabhängige Community-Projekt steht in keiner Verbindung zu CKW oder zum Home-Assistant-Projekt und wird von diesen weder unterstützt noch offiziell bestätigt.

## Inhaltsverzeichnis

- [Voraussetzungen](#voraussetzungen)
- [1. CKW-Tarifdaten-Package einbinden](#1-ckw-tarifdaten-package-einbinden)
- [2. Template-Sensor für den aktuellen Preis](#2-template-sensor-für-den-aktuellen-preis)
- [3. Acht Zeitfenster-Sensoren erstellen](#3-acht-zeitfenster-sensoren-erstellen)
- [4. Preisempfehlung für heute und morgen](#4-preisempfehlung-für-heute-und-morgen)
- [5. Automationsbeispiel](#5-automationsbeispiel)
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

## 3. Acht Zeitfenster-Sensoren erstellen

Erstelle acht Template-Sensor-Helfer. Für jeden Helfer verwendest du denselben Zustandscode weiter unten, aber passt die drei Einstellungen am Anfang entsprechend der Tabelle an. Der Code filtert erst auf den Zieldatumsbereich und sortiert danach. Die Sensoren für den zweiten Platz schliessen Zeitfenster aus, die das jeweils beste Zeitfenster überlappen.

| Helfername | `tage_voraus` | `teuerstes` | `zweites` |
|---|---:|---|---|
| CKW günstigstes Zeitfenster von 2h | `0` | `false` | `false` |
| CKW zweitgünstigstes Zeitfenster von 2h | `0` | `false` | `true` |
| CKW teuerstes Zeitfenster von 2h | `0` | `true` | `false` |
| CKW zweitteuerstes Zeitfenster von 2h | `0` | `true` | `true` |
| CKW günstigstes Zeitfenster morgen von 2h | `1` | `false` | `false` |
| CKW zweitgünstigstes Zeitfenster morgen von 2h | `1` | `false` | `true` |
| CKW teuerstes Zeitfenster morgen von 2h | `1` | `true` | `false` |
| CKW zweitteuerstes Zeitfenster morgen von 2h | `1` | `true` | `true` |

Für jeden Sensor: Code kopieren, die drei Parameterzeilen oben setzen, den Code im Feld **Zustand** des jeweiligen Template-Sensor-Helfers speichern. Nicht unter **Entwicklerwerkzeuge → Zustände → Zustand setzen** einfügen; dieses Werkzeug wertet Jinja nicht aus.

```jinja
{% set tage_voraus = 0 %}
{% set teuerstes = false %}
{% set zweites = false %}

{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set zieltag = (now() + timedelta(days=tage_voraus)).strftime('%Y-%m-%d') %}
{% set ns = namespace(fenster=[]) %}

{# Acht Viertelstundenpreise ergeben ein 2-Stunden-Fenster #}
{% for i in range(0, (p | count) - 7) %}
  {% set block = p[i:i+8] %}
  {% set werte = namespace(liste=[]) %}

  {% for q in block %}
    {% if q.integrated is defined and q.integrated | count > 0 %}
      {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
    {% endif %}
  {% endfor %}

  {% if block | count == 8
        and werte.liste | count == 8
        and block[0].start_timestamp[:10] == zieltag
        and block[7].end_timestamp[:10] == zieltag %}
    {% set ns.fenster = ns.fenster + [{
      'start': i,
      'preis': (werte.liste | sum) / 8
    }] %}
  {% endif %}
{% endfor %}

{% if teuerstes %}
  {% set sortiert = ns.fenster | sort(attribute='preis', reverse=true) %}
{% else %}
  {% set sortiert = ns.fenster | sort(attribute='preis') %}
{% endif %}

{% set auswahl = namespace(fenster=none) %}

{% if sortiert | count > 0 %}
  {% if zweites %}
    {% set bestes = sortiert[0] %}
    {% for f in sortiert %}
      {% if auswahl.fenster is none
            and (f.start >= bestes.start + 8
                 or f.start + 8 <= bestes.start) %}
        {% set auswahl.fenster = f %}
      {% endif %}
    {% endfor %}
  {% else %}
    {% set auswahl.fenster = sortiert[0] %}
  {% endif %}
{% endif %}

{% if auswahl.fenster is not none %}
  {% set i = auswahl.fenster.start %}
  {{ p[i].start_timestamp[:10] }} · {{ p[i].start_timestamp[11:16] }}–{{ p[i+7].end_timestamp[11:16] }} · {{ '%.2f' | format(auswahl.fenster.preis * 100) }} Rp./kWh
{% else %}
Für den gewählten Tag wurden keine passenden CKW-Preise gefunden
{% endif %}
```

Die `sensor.`-IDs entstehen aus den Helfernamen. Home Assistant kann Umlaute umschreiben oder bei Namenskonflikten eine Endung hinzufügen. Prüfe die tatsächlichen IDs unter **Einstellungen → Geräte & Dienste → Entitäten** und verwende genau diese IDs in weiteren Vorlagen.

## 4. Preisempfehlung für heute und morgen

Für jede Empfehlung einen weiteren Template-Sensor-Helfer erstellen, zum Beispiel `CKW Preisempfehlung` und `CKW Preisempfehlung morgen`. Der folgende Code berechnet die vier Zeitfenster direkt aus `sensor.ckw_tarifdaten`; er ist dadurch nicht von den acht separaten Zeitfenster-Helfern abhängig.

Für die Empfehlung **heute** `tage_voraus` auf `0` setzen. Für die Empfehlung **morgen** auf `1` setzen.

```jinja
{% set tage_voraus = 0 %}
{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set zieltag = (now() + timedelta(days=tage_voraus)).strftime('%Y-%m-%d') %}
{% set ns = namespace(fenster=[]) %}

{% for i in range(0, (p | count) - 7) %}
  {% set block = p[i:i+8] %}
  {% set werte = namespace(liste=[]) %}

  {% for q in block %}
    {% if q.integrated is defined and q.integrated | count > 0 %}
      {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
    {% endif %}
  {% endfor %}

  {% if block | count == 8
        and werte.liste | count == 8
        and block[0].start_timestamp[:10] == zieltag
        and block[7].end_timestamp[:10] == zieltag %}
    {% set ns.fenster = ns.fenster + [{
      'start': i,
      'preis': (werte.liste | sum) / 8
    }] %}
  {% endif %}
{% endfor %}

{% set billig = ns.fenster | sort(attribute='preis') %}
{% set teuer = ns.fenster | sort(attribute='preis', reverse=true) %}
{% set auswahl = namespace(billig_zwei=none, teuer_zwei=none) %}

{% if billig | count > 0 %}
  {% set bestes = billig[0] %}
  {% for f in billig %}
    {% if auswahl.billig_zwei is none
          and (f.start >= bestes.start + 8
               or f.start + 8 <= bestes.start) %}
      {% set auswahl.billig_zwei = f.start %}
    {% endif %}
  {% endfor %}
{% endif %}

{% if teuer | count > 0 %}
  {% set bestes = teuer[0] %}
  {% for f in teuer %}
    {% if auswahl.teuer_zwei is none
          and (f.start >= bestes.start + 8
               or f.start + 8 <= bestes.start) %}
      {% set auswahl.teuer_zwei = f.start %}
    {% endif %}
  {% endfor %}
{% endif %}

{% if billig | count > 0
      and auswahl.billig_zwei is not none
      and teuer | count > 0
      and auswahl.teuer_zwei is not none %}
  {% set b = [billig[0].start, auswahl.billig_zwei] | sort %}
  {% set t = [teuer[0].start, auswahl.teuer_zwei] | sort %}
{{ 'Morgen' if tage_voraus == 1 else 'Heute' }} ({{ zieltag }}): Günstig {{ p[b[0]].start_timestamp[11:16] }}–{{ p[b[0]+7].end_timestamp[11:16] }} und {{ p[b[1]].start_timestamp[11:16] }}–{{ p[b[1]+7].end_timestamp[11:16] }}. Teuer {{ p[t[0]].start_timestamp[11:16] }}–{{ p[t[0]+7].end_timestamp[11:16] }} und {{ p[t[1]].start_timestamp[11:16] }}–{{ p[t[1]+7].end_timestamp[11:16] }}.
{% else %}
Für {{ 'morgen' if tage_voraus == 1 else 'heute' }} konnten nicht genügend CKW-Preise für vier nicht überlappende Zeitfenster gefunden werden.
{% endif %}
```

Der Ausgabezustand ist kurz gehalten. Die lesbare Darstellung der Empfehlungen ist in der zentralen [Dashboard-Dokumentation](dashboard.md) beschrieben.

Für 5-Stunden-Fenster zum Laden eines Elektroautos siehe die separate [Anleitung zu Auto-Laden-Zeitfenstern](auto-laden.md).

## 5. Automationsbeispiel

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

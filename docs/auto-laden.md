# Auto-Laden-Zeitfenster

Diese Anleitung ergänzt die CKW-Preisfenster um zwei Template-Sensor-Helfer für ein Elektroauto:

- **Auto laden** ermittelt das günstigste zusammenhängende Zeitfenster von fünf Stunden.
- **Auto laden gesperrt** ermittelt das teuerste zusammenhängende Zeitfenster von fünf Stunden, das sich nicht mit „Auto laden“ überschneidet.

Ein 5-Stunden-Fenster besteht aus 20 aufeinanderfolgenden CKW-Viertelstundenwerten. Gesucht wird im Zeitraum von heute 12:00 Uhr bis morgen 12:00 Uhr. Verwendet werden ausschliesslich Preise, die im Attribut `prices` von `sensor.ckw_tarifdaten` vorhanden sind. Solange kein gültiger Preis für morgen vorhanden ist, zeigen beide Sensoren `warte auf Preise von morgen`. Sobald Preise für morgen vorliegen, berechnen sie die 5-Stunden-Fenster aus den verfügbaren Preisen.

Beide Sensoren sind Template-Sensor-Helfer. Erstelle sie unter **Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Template → Sensor**. Füge jeweils nur den Jinja-Code ins Feld **Zustand** ein. Lasse Geräteklasse und Einheit leer, da der Zustand den Zeitraum und den Durchschnittspreis als Text enthält. Prüfe die erzeugten Entity-IDs in Home Assistant und passe die Referenz im zweiten Sensor bei Bedarf an.

Die Karten für diese beiden Sensoren stehen in den [Strompreis-Karten](dashboard.md#strompreis-karten). Die Paket-Einrichtung steht unter [1. CKW-Tarifdaten-Package einbinden](anleitung.md#1-ckw-tarifdaten-package-einbinden).

## Inhaltsverzeichnis

- [Günstigstes 5-Stunden-Fenster](#günstigstes-5-stunden-fenster)
- [Teuerstes nicht überlappendes 5-Stunden-Fenster](#teuerstes-nicht-überlappendes-5-stunden-fenster)
- [1. CKW-Tarifdaten-Package einbinden](anleitung.md#1-ckw-tarifdaten-package-einbinden)
- [Strompreis-Karten](dashboard.md#strompreis-karten)

## Günstigstes 5-Stunden-Fenster

Name: `Auto laden`

```jinja
{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set morgen = (today_at('00:00') + timedelta(days=1)).strftime('%Y-%m-%d') %}
{% set morgen_preise = namespace(vorhanden=false) %}
{% for q in p %}
  {% if q.start_timestamp is defined and q.start_timestamp[:10] == morgen
        and q.integrated is defined and q.integrated | count > 0 %}
    {% set morgen_preise.vorhanden = true %}
  {% endif %}
{% endfor %}
{% set start_limit = as_timestamp(today_at('12:00')) %}
{% set end_limit = start_limit + 86400 %}
{% set ns = namespace(fenster=[]) %}

{% if not morgen_preise.vorhanden %}
warte auf Preise von morgen
{% else %}
{% for i in range(0, (p | count) - 19) %}
  {% set block = p[i:i+20] %}
  {% set werte = namespace(liste=[]) %}
  {% for q in block %}
    {% if q.integrated is defined and q.integrated | count > 0 %}
      {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
    {% endif %}
  {% endfor %}
  {% if block | count == 20 and werte.liste | count == 20
        and as_timestamp(block[0].start_timestamp, 0) >= start_limit
        and as_timestamp(block[19].end_timestamp, 0) <= end_limit %}
    {% set ns.fenster = ns.fenster + [{
      'start': i,
      'preis': (werte.liste | sum) / 20
    }] %}
  {% endif %}
{% endfor %}

{% set sortiert = ns.fenster | sort(attribute='preis') %}
{% if sortiert | count > 0 %}
  {% set i = sortiert[0].start %}
  {{ p[i].start_timestamp[:10] }} · {{ p[i].start_timestamp[11:16] }}–{{ p[i+19].end_timestamp[11:16] }} · {{ '%.2f' | format(sortiert[0].preis * 100) }} Rp./kWh
{% else %}
Für heute 12:00 bis morgen 12:00 wurden keine vollständigen 5-Stunden-Fenster mit verfügbaren CKW-Preisen gefunden
{% endif %}
{% endif %}
```

## Teuerstes nicht überlappendes 5-Stunden-Fenster

Name: `Auto laden gesperrt`

Der Sensor liest den Zeitraum aus `sensor.auto_laden` und lässt jedes 5-Stunden-Fenster aus, das diesen Zeitraum auch nur teilweise überlappt. Direkt aneinandergrenzende Fenster sind erlaubt, beispielsweise 12:00–17:00 und 17:00–22:00.

```jinja
{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set guenstig = states('sensor.auto_laden') %}
{% set morgen = (today_at('00:00') + timedelta(days=1)).strftime('%Y-%m-%d') %}
{% set morgen_preise = namespace(vorhanden=false) %}
{% for q in p %}
  {% if q.start_timestamp is defined and q.start_timestamp[:10] == morgen
        and q.integrated is defined and q.integrated | count > 0 %}
    {% set morgen_preise.vorhanden = true %}
  {% endif %}
{% endfor %}
{% set start_limit = as_timestamp(today_at('12:00')) %}
{% set end_limit = start_limit + 86400 %}
{% set ns = namespace(fenster=[]) %}

{% if not morgen_preise.vorhanden %}
warte auf Preise von morgen
{% elif guenstig not in ['unknown', 'unavailable', 'none', ''] %}
  {% set teile = guenstig.split(' · ') %}
  {% set guenstig_zeiten = teile[1].split('–') %}
  {% set guenstig_start = as_timestamp(as_datetime(teile[0] ~ 'T' ~ guenstig_zeiten[0] ~ ':00')) %}
  {% set guenstig_ende = as_timestamp(as_datetime(teile[0] ~ 'T' ~ guenstig_zeiten[1] ~ ':00')) %}

  {% for i in range(0, (p | count) - 19) %}
    {% set block = p[i:i+20] %}
    {% set werte = namespace(liste=[]) %}
    {% for q in block %}
      {% if q.integrated is defined and q.integrated | count > 0 %}
        {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
      {% endif %}
    {% endfor %}
    {% if block | count == 20 and werte.liste | count == 20
          and as_timestamp(block[0].start_timestamp, 0) >= start_limit
          and as_timestamp(block[19].end_timestamp, 0) <= end_limit %}
      {% set fenster_start = as_timestamp(block[0].start_timestamp, 0) %}
      {% set fenster_ende = as_timestamp(block[19].end_timestamp, 0) %}
      {% if fenster_ende <= guenstig_start or fenster_start >= guenstig_ende %}
        {% set ns.fenster = ns.fenster + [{
          'start': i,
          'preis': (werte.liste | sum) / 20
        }] %}
      {% endif %}
    {% endif %}
  {% endfor %}

  {% set sortiert = ns.fenster | sort(attribute='preis', reverse=true) %}
  {% if sortiert | count > 0 %}
    {% set i = sortiert[0].start %}
    {{ p[i].start_timestamp[:10] }} · {{ p[i].start_timestamp[11:16] }}–{{ p[i+19].end_timestamp[11:16] }} · {{ '%.2f' | format(sortiert[0].preis * 100) }} Rp./kWh
  {% else %}
  Kein nicht überlappendes 5-Stunden-Fenster mit verfügbaren CKW-Preisen gefunden
  {% endif %}
{% else %}
Warte auf das Zeitfenster von Auto laden
{% endif %}
```

Die Ausgabe folgt dem bestehenden Format der CKW-Zeitfenster: Datum, Start- und Endzeit sowie durchschnittlicher Preis in Rp./kWh. Beispiel einer Messung am 22.11.2026: `2026-11-22 · 12:00–17:00 · 19.03 Rp./kWh` und `2026-11-22 · 17:00–22:00 · 24.98 Rp./kWh`. Diese Beispielwerte sind Momentaufnahmen. Der Mittelwert wird aus den 20 Viertelstundenpreisen berechnet. Eine Anzeige wie `12:00–17:00` und `17:00–22:00` ist nicht überlappend.

Wenn einer der Sensoren keinen passenden Zustand zeigt, zuerst in **Entwicklerwerkzeuge → Zustände** prüfen, ob `sensor.ckw_tarifdaten` das Attribut `prices` enthält. Im Template-Editor kann die Preisanzahl mit `{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}{{ p | count }}` kontrolliert werden. Prüfe ausserdem, ob der günstige Sensor tatsächlich die Entity-ID `sensor.auto_laden` hat.

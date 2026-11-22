# 2-Stunden-Zeitfenster

Diese Anleitung richtet die acht Template-Sensor-Helfer für die günstigsten und teuersten 2-Stunden-Zeitfenster von heute und morgen ein. Voraussetzung ist das in der [Anleitung](anleitung.md#1-ckw-tarifdaten-package-einbinden) beschriebene CKW-Tarifdaten-Package mit `sensor.ckw_tarifdaten`.

## Inhaltsverzeichnis

- [Acht Zeitfenster-Sensoren erstellen](#acht-zeitfenster-sensoren-erstellen)
- [Weiterführende Anleitungen](#weiterführende-anleitungen)

## Acht Zeitfenster-Sensoren erstellen

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
## Weiterführende Anleitungen

- [Preisempfehlungen für heute und morgen](preisempfehlungen.md)
- [Dashboard-Karten für Zeitfenster](dashboard.md#vier-zeitfenster-für-heute)
- [Auto-Laden-Zeitfenster](auto-laden.md)
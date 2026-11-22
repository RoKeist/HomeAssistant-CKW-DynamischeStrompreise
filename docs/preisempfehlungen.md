# Preisempfehlungen für heute und morgen

Diese Anleitung richtet die zwei Template-Sensor-Helfer ein, die je zwei günstige und teure, nicht überlappende 2-Stunden-Fenster empfehlen. Voraussetzung ist das in der [Anleitung](anleitung.md#1-ckw-tarifdaten-package-einbinden) beschriebene CKW-Tarifdaten-Package mit `sensor.ckw_tarifdaten`.

## Inhaltsverzeichnis

- [Preisempfehlung für heute und morgen](#preisempfehlung-für-heute-und-morgen)
- [Weiterführende Anleitungen](#weiterführende-anleitungen)

## Preisempfehlung für heute und morgen

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
## Weiterführende Anleitungen

- [Hoch- & Tiefpreis Zeitfenster](zeitfenster-2h.md)
- [Dashboard-Karten für Zeitfenster und Empfehlungen](dashboard.md#strompreis-karten)
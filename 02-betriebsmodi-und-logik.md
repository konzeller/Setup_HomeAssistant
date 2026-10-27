# 2. Betriebsmodi in Home Assistant umsetzen

Diese Datei legt fest, wie Home Assistant den aktiven Sollwert auswählt und wie die Geräte zusammenarbeiten. Die konkreten Automationen liegen im [YAML-Package](packages/klima_dachgeschoss.yaml). Alle `replace_me`-Entity-IDs dort müssen vor dem Laden ersetzt werden.

## 2.1 Vorbedingungen und effektive Betriebsart

Verwende die Helfer aus [Schritt 1](01-entitaeten-und-helfer.md). Das YAML-Package erzeugt `binary_sensor.klima_urlaub_aktiv` und wertet Start- und Enddatum aus. Erstelle diesen Sensor nicht zusätzlich in der UI. Er verwendet folgende Template-Logik:

1. Öffne **Einstellungen > Geräte & Dienste > Helfer > Helfer erstellen > Template > Template-Binärsensor**.
2. Name: `Klima Urlaub aktiv`; Entity-ID: `binary_sensor.klima_urlaub_aktiv`.
3. Zustand (Datums- und Zeithelfer müssen Datum **und** Uhrzeit enthalten; setze das Urlaubsende auf 23:59 des letzten Urlaubstags):

```jinja
{% set start = as_timestamp(states('input_datetime.klima_urlaub_start'), default=none) %}
{% set ende = as_timestamp(states('input_datetime.klima_urlaub_ende'), default=none) %}
{{ is_state('input_boolean.klima_urlaub_aktiv', 'on')
   and start is not none and ende is not none
   and start <= as_timestamp(now()) <= ende }}
```

4. Der Template-Sensor wird durch Home Assistant und den Zeitwechsel neu ausgewertet.

Ein `person`-Entity wird als Zuhause gewertet, wenn mindestens eine Person zuhause ist. Das Package erzeugt `group.haushalt`; ersetze dort `person.replace_me_person_1` durch deine Person-Entity und ergänze weitere Bewohner bei Bedarf. Die Gruppe ist zuhause, wenn mindestens eine Person zuhause ist.

**Priorität für Sollwerte:** Urlaub → abwesend → manuelle Übersteuerung bei Anwesenheit → Zuhause nach Zeitplan. Sicherheitsaktionen (z. B. Fenster bei Regen schließen) laufen unabhängig davon und dürfen immer eingreifen. Fensteröffnen hat zusätzlich eine eigene harte Urlaubs- und Sicherheitsbedingung.

## 2.2 Heizperiode als abgeleiteter Zustand

Der Außentemperaturfühler am Südbalkon ist sonnenexponiert. Nimm für die saisonale Heizfreigabe bevorzugt einen Außentemperatursensor der Wetterintegration.

1. Unter **Einstellungen > Geräte & Dienste > Helfer > Helfer erstellen > Statistik** den Wetter-Außentemperatursensor wählen.
2. Statistiktyp `Mittelwert` wählen und das maximale Alter auf 24 Stunden setzen. Beispiel-ID: `sensor.klima_aussentemperatur_24h_mittel`.
3. Zwei Automationen (Schritt 3.2) schalten `input_boolean.klima_heizperiode` mit Hysterese:
   - Heizperiode an, wenn der Mittelwert unter `input_number.klima_heizperiode_ein` (Start 15 °C) für 48 Stunden bleibt.
   - Heizperiode aus, wenn er über `input_number.klima_heizperiode_aus` (Start 17 °C) für 48 Stunden bleibt.
4. Beim Home-Assistant-Start den Mittelwert erneut bewerten. Wenn der Statistikwert unbekannt ist oder noch keine 24 Stunden Historie hat, Heizperiode nicht automatisch abschalten; bis ausreichend Daten vorhanden sind manuell entscheiden.

Das ist eine Freigabe, kein Heizbefehl. Innerhalb der Heizperiode wird nur geheizt, wenn der aktuelle Raum-Sollwert es verlangt. Außerhalb erhalten die Thermostate einen niedrigen Schutz-Sollwert statt `off`.

## 2.3 Heizkörper und Daikin als getrennte Wärmequellen

Die drei Wohnbereichs-DECT-301 steuern Heizkörper an der Zentralheizung; die Wärmequelle im Mehrfamilienhaus ist nicht per Home Assistant schaltbar. Es gibt daher keine belastbare automatische Auswahl nach Kosten. Beginne mit dieser festen Rollenverteilung:

- Heizkörper regeln den Grundbedarf im Wohn-/Esszimmer und in der Küche.
- Bad-DECT-301 regelt sein eigenes Zeitprofil.
- Daikin heizt ausschließlich als Komfort-Boost bei Anwesenheit und deutlicher Sollwertabweichung.
- Schlafzimmer erhält erst eine automatische Heizregelung, wenn dort der angekündigte DECT 301 installiert und in HA verfügbar ist.

Für den Wohnbereich verwendet die Regelung den AVM-440-Fühler auf Aufenthaltsniveau. Der höher sitzende Daikin-Fühler dient zur Erkennung einer Temperaturschichtung. Die Daikin darf im Boost nur starten, wenn alle Fenster geschlossen sind, die Sensoren gültig sind und die Temperatur mindestens 1,5 °C für 15 Minuten unter Soll liegt. Sie beendet den Boost bei erreichter Zieltemperatur (mit Hysterese), geöffnetem Fenster, Abwesenheit/Urlaub oder nach einer maximalen Laufzeit. Die Heizkörper zunächst nicht absenken; erst nach Vergleich von Temperaturverlauf und Verbrauch testen.

## 2.4 Daikin-Zustände und Moduswechsel

Vor dem Anlegen der Automationen in **Entwicklerwerkzeuge > Zustände** bei der Daikin prüfen:

- `state_attr('climate.DEINE_DAIKIN', 'hvac_modes')` enthält die tatsächlich unterstützten Werte `heat`, `cool`, `fan_only` und `off`.
- `state_attr('climate.DEINE_DAIKIN', 'hvac_action')` unterscheidet beispielsweise `heating`, `cooling` und `idle`.
- Der Sollwert wird über `climate.set_temperature` gesetzt.

Nur Modi verwenden, die das Gerät meldet. Die Daikin darf nicht gleichzeitig von mehreren unabhängigen Automationen gesteuert werden. Für die erste Umsetzung bekommt jede Betriebsart klare Sperrbedingungen:

1. Fenster offen oder Daikin nicht verfügbar: Daikin aus.
2. Kühlbedarf: `cool` hat Vorrang vor Lüfter und Heiz-Boost.
3. Heiz-Boost: nur bei Anwesenheit, Heizperiode und klarer Temperaturabweichung.
4. Ventilator-Durchmischung: nur wenn Daikin in `heat` steht, `hvac_action` `idle` ist und die Temperaturdifferenz groß genug ist.
5. Sonst keine Aktion beziehungsweise im Urlaubsmodus Daikin aus.

Nach erfolgreichem Test sollten diese Aktionen in **einer Daikin-Steuerungsautomation mit `choose`** zusammengeführt werden. Bis dahin die unten beschriebenen Regeln so anlegen, dass sie sich gegenseitig durch Bedingungen ausschließen.

## 2.5 Luftumwälzung im Heizbetrieb

Der `fan_only`-Modus wärmt nicht, sondern bewegt Raumluft. Die Regelung startet ihn, wenn Daikin-Temperatur minus AVM-440-Temperatur mindestens 2 °C für 10 Minuten beträgt. Voraussetzungen:

- Heizperiode aktiv und jemand zuhause.
- Alle Wohnbereichsfenster geschlossen; beide Temperaturwerte numerisch/verfügbar.
- Daikin steht auf `heat`, ist aber gerade nicht aktiv am Heizen (`hvac_action: idle`). Kühlbetrieb nie unterbrechen.
- `fan_only` wird 10 Minuten betrieben; danach `heat` und der zuvor gültige Sollwert wiederherstellen.
- Mindestens 30 Minuten Pause bis zum nächsten Lauf. Automation `mode: single` verwenden und eine Start-/Abbruch-Sicherung ergänzen, damit der Ventilator nach Neustart nicht dauerhaft eingeschaltet bleibt.

Zunächst eine Woche nur die Temperaturdifferenz protokollieren. Bei Sensorabweichungen zuerst Fühler vergleichen/kalibrieren, nicht einfach die Schwelle stark reduzieren.

## 2.6 Lüften: Fenster nur mit verifizierten Schutzbedingungen öffnen

Es gibt keinen lokalen Windsensor. Deshalb ist die Wettervorhersage eine notwendige, aber keine gleichwertige Windsicherung. Wenn aktuelle Wind-/Böenwerte nicht verfügbar, veraltet oder für den Standort unplausibel sind, Fensterautomation nur schließen, niemals öffnen lassen.

Öffnungsbedingungen für einen Raum müssen alle zutreffen:

- Urlaub aus; Fenster geschlossen; Haus anwesend **oder** normale Abwesenheit (beides erlaubt, aber begrenzte Lüftungsdauer).
- Raum ist über seinem Sollwert und Außenluft mindestens 2 °C kühler.
- Wetter-Entity und Sicherheitswerte verfügbar; kein Regen; Wind/Böen unter dem konservativen, vom Fensterhersteller zulässigen Grenzwert.
- Öffnung nur für ein Fenster je Raum und für begrenzte Zeit.

Sofort schließen bei Regen, Windlimitüberschreitung, fehlenden Schutzdaten, Urlaubsbeginn oder Ablauf des Lüftungstimers. Das Schließen ist im Urlaub ausdrücklich weiterhin erlaubt. Vor Öffnung Daikin pausieren; bei Fensterzustand unbekannt Daikin nicht einschalten.

## 2.7 Rollläden und Wettervorhersage

Für jede Fassadenseite eine eigene Gruppe anlegen. Aktuell sind die vier Wohnbereichsfenster westorientiert; die zwei Schlafzimmerfenster sowie mögliche künftige Ostfenster nicht in die Westgruppe aufnehmen.

- West: bei erwarteter Nachmittagssonne und warmem Tagesmaximum frühzeitig beschatten.
- Ost (falls ergänzt): bei erwarteter Morgensonne beschatten.
- Winter: bei Sonneneinstrahlung Rollläden öffnen, wenn kein Komfort-/Privatsphäregrund dagegenspricht.
- Urlaub sperrt das automatische **Öffnen von Fenstern**, nicht die Beschattung.

Die Vorhersageintegration kann sich je nach Anbieter in Feldern und Aktualisierung unterscheiden. In **Entwicklerwerkzeuge > Aktionen** `weather.get_forecasts` mit deiner Wetter-Entity und Typ `hourly` testen. Die Aktionsantwort enthält je nach Integration nicht zwingend Böen oder Einstrahlung. Nur tatsächlich gelieferte und aktuelle Felder in Templates verwenden; andernfalls Beschattung zunächst nach Sonnenstand, Uhrzeit und realer Innentemperatur steuern.

## 2.8 Kühlung und Abwesenheitsgrenze

Kühlen im Abwesenheitsmodus, wenn der AVM 440 die Einschaltgrenze (Start 25,5 °C) erreicht, Fenster geschlossen und Daikin/Temperatur verfügbar sind. Abschalten erst bei der niedrigeren Ausschaltgrenze (Start 24,5 °C), damit die Regelung nicht taktet. `input_number.klima_abwesenheit_max` bleibt die gewünschte Obergrenze 26 °C; weil Räume und Sensoren träge sind, muss die Kühlung darunter anlaufen. Das ist keine Garantie, dass 26 °C bei jeder Außentemperatur eingehalten werden können.

Bei Anwesenheit nur im Komfortzeitplan und mit `input_number.klima_wohnen_kuehlziel` kühlen. Im Urlaub standardmäßig keine Komfortkühlung; wenn die Anlage zum Gebäudeschutz bei extremer Hitze laufen soll, dafür eine separate, konservative Maximaltemperatur festlegen.

Beschattung wird bei prognostizierter Hitze vorgezogen. Für die Temperaturregelung bleibt der AVM 440 auf Aufenthaltsniveau maßgeblich; die Daikin-Fühlerhöhe von 2,5 m kann zu Abweichungen führen und muss bei der Inbetriebnahme kalibriert werden.
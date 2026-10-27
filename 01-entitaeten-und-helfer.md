# 1. Entitäten erfassen und Helfer anlegen

Diese Anleitung verwendet sprechende Beispiel-IDs. Home Assistant erzeugt IDs abhängig von Integration und Gerätenamen; die Beispiele müssen daher mit den tatsächlichen IDs abgeglichen werden.

## 1.1 Geräte-Entitäten inventarisieren

Öffne **Entwicklerwerkzeuge > Zustände** und suche nacheinander nach den Geräten. Notiere Entity-ID, Zustand und verfügbare Attribute. Für Automationen müssen Entitäten außerdem in **Einstellungen > Geräte & Dienste** erreichbar und verfügbar sein.

| Funktion | Erwarteter Entitätstyp | Beispiel-ID / Hinweis |
|---|---|---|
| Anwesenheit | `person` oder Gruppe | `person.<name>`; für mehrere Bewohner eine Gruppe aller Personen verwenden |
| Wetter | `weather` | `weather.home`; prüfen, ob die Aktion `weather.get_forecasts` funktioniert |
| Außentemperatur | `sensor` | Wetterintegration bevorzugen, nicht den sonnenexponierten Südbalkonfühler |
| AVM 440 Temperatur und Feuchte | `sensor` | Wohnbereich auf Aufenthaltsniveau; beide Messwerte notieren |
| Daikin Temperatur und Feuchte | `sensor` oder Attribute der `climate`-Entität | Messhöhe ca. 2,5 m; nicht als alleinige Raumreferenz verwenden |
| Daikin Steuerung | `climate` | `climate.daikin_wohnzimmer`; unterstützte HVAC-Modi und Attribute prüfen |
| Schlafzimmer Raumklima | `sensor` | Temperatur und Feuchte des Velux-Sensors |
| Bad Raumtemperatur | `climate`-Attribut oder `sensor` | Messwert des Bad-DECT-301 prüfen |
| Heizkörperthermostate | `climate` | drei einzelne Wohnbereichs-Entitäten plus Bad; Sollwertsteuerung testen |
| Dachfenster | je nach Integration `cover` und/oder `binary_sensor` | Öffnen/Schließen, Position, Kontaktzustand und Verfügbarkeit einzeln prüfen |
| Regenmelder | `binary_sensor` | pro Fenster zuordnen; Zustand `on` muss als Regen/Nässe verifiziert werden |
| Rollläden | `cover` | je Fenster zuordnen; prüfen, ob nur auf/ab oder auch Position unterstützt wird |

Lege eine Zuordnungstabelle mit Raum, Fenster, Regensensor, Fenster-Entity und Rollladen-Entity an. Fensterantriebe und Rollläden können in der Integration unterschiedliche Entity-IDs haben. Nicht annehmen, dass ein Dachfenster über `cover.set_cover_position` Prozentpositionen unterstützt.

**Fenster-Sicherheitsprüfung:** Prüfe bei der Wetter-Entity das Attribut `wind_speed` und in der stündlichen Vorhersage das Feld `wind_speed`. Das ist kein lokaler Windsensor und keine Böenmessung. Automatisches Öffnen bleibt gesperrt, falls aktuelle oder vorhergesagte Windwerte fehlen, veraltet sind oder die Velux-Grenze nicht sicher eingehalten werden kann. Der Regensensor allein schützt ein Dachfenster nicht vor Wind.

## 1.2 Helfer anlegen

Bei Verwendung von [packages/klima_dachgeschoss.yaml](packages/klima_dachgeschoss.yaml) werden Eingabeauswahl, Datumsfelder, Schalter, Timer und Zahlen-Helfer durch YAML angelegt. Diese nicht zusätzlich in der UI erzeugen. Die Zahlen-Helfer erhalten nach dem ersten Laden den jeweils passenden Betriebswert über **Einstellungen > Geräte & Dienste > Helfer**. Ohne Package können die unten aufgeführten Helfer manuell erstellt werden. Die Zeitplan-Helfer werden in beiden Varianten in der UI eingerichtet.

| Helfer | Typ | Vorschlags-ID | ID Implementiert | Zweck / Startwert |
|---|---|---|---|---|
| Klima-Automatik | Eingabeauswahl | `input_select.klima_betriebsart` | `input_select.ha_klima_betriebsart` | Optionen `Automatik`, `Manuell`; Urlaub wird aus Datumshelfern berechnet |
| Urlaubsplan aktiv | Eingabe-Schalter | `input_boolean.klima_urlaub_aktiv` | `input_boolean.ha_urlaub` | Schaltet den eingetragenen Urlaubszeitraum scharf |
| Heizperiode aktiv | Eingabe-Schalter | `input_boolean.klima_heizperiode` | `input_boolean.ha_klima_heizperiode` | Package-Status mit 48-Stunden-Hysterese; beim ersten Winterstart ggf. manuell einschalten |
| Urlaubsbeginn | Eingabedatum/-zeit | `input_datetime.klima_urlaub_start` | `input_datetime.ha_urlaubsbeginn` | Datum aktivieren; Uhrzeit optional |
| Urlaubsende | Eingabedatum/-zeit | `input_datetime.klima_urlaub_ende` | `input_datetime.ha_urlaubsende` | Datum und Uhrzeit aktivieren; letzten Urlaubstag bis 23:59 einschließen |
| Manuelle Übersteuerung | Timer | `timer.klima_manuelle_uebersteuerung` | `timer.ha_klima_manuelle_ubersteuerung` | Dauer 2 Stunden; beim Ablauf zurück auf Automatik |
| Wohnbereich Komfort | Eingabezahl | `input_number.klima_wohnen_komfort` | `input_number.ha_komfort_temperatur_wohnzimmer` | Startwert z. B. 20 °C |
| Wohnbereich Eco | Eingabezahl | `input_number.klima_wohnen_eco` | `input_number.ha_eco_temperatur_wohnzimmer` | Startwert z. B. 17 °C |
| Schlafzimmer Komfort | Eingabezahl | `input_number.klima_schlafen_komfort` | `input_number.ha_komfort_temperatur_schlafzimmer` | Startwert z. B. 18 °C |
| Schlafzimmer Nacht | Eingabezahl | `input_number.klima_schlafen_nacht` | `input_number.ha_nacht_temperatur_schlafzimmer` | Startwert z. B. 17 °C |
| Bad Komfort | Eingabezahl | `input_number.klima_bad_komfort` | `input_number.ha_komfort_temperatur_bad` | Startwert z. B. 20 °C |
| Bad Eco | Eingabezahl | `input_number.klima_bad_eco` | `input_number.ha_eco_temperatur_bad` | Startwert z. B. 17 °C |
| Schutz-Sollwert | Eingabezahl | `input_number.klima_schutz_sollwert` | `input_number.ha_soll_klima_schutz` | Startwert z. B. 12 °C; niedriger Heiz-Sollwert außerhalb der Heizperiode, kein Ersatz für Gebäudefrostschutz |
| Urlaub Sollwert | Eingabezahl | `input_number.klima_urlaub_sollwert` | `input_number.ha_soll_temperatur_urlaub` | Startwert z. B. 16 °C; kein Frostschutz für Leitungen außerhalb der Wohnung |
| Abwesenheit Kühlgrenze | Eingabezahl | `input_number.klima_abwesenheit_max` | `input_number.ha_maximal_temperatur_abwesenheit` | 26 °C, wie vorgegeben |
| Wohnbereich Kühlziel | Eingabezahl | `input_number.klima_wohnen_kuehlziel` | `input_number.ha_kuhlziel_wohnzimmer` | Startwert z. B. 25 °C; manueller Komfortwert Zuhause |
| Kühlung einschalten | Eingabezahl | `input_number.klima_kuehlung_ein` | `number.ha_temperatur_kuhlung_ein` | Startwert 25,5 °C, damit die Regelung vor 26 °C reagieren kann |
| Kühlung ausschalten | Eingabezahl | `input_number.klima_kuehlung_aus` | `number.ha_temperatur_kuhlung_aus` | Startwert 24,5 °C; muss unter dem Einschaltwert liegen |
| Prognose-Kühlgrenze | Eingabezahl | `input_number.klima_kuehlung_vorhersage` | `input_number.ha_temperatur_kuhlung_vorhersage` | Startwert z. B. 30 °C als Außentemperatur-Prognose für die nächsten 6 Stunden |
| Heizperiode ein | Eingabezahl | `input_number.klima_heizperiode_ein` | `input_number.ha_temperatur_heizperiode_ein` | Startwert 15 °C Außentemperatur |
| Heizperiode aus | Eingabezahl | `input_number.klima_heizperiode_aus` | `input_number.klima_heizperiode_aus` | Startwert 17 °C; größer als Einschaltwert für Hysterese |
| Windlimit fürs Lüften | Eingabezahl | `input_number.klima_max_windgeschwindigkeit` | `input_number.ha_zulassige_windgeschwindigkeit` | Wetter-Entity und Forecast-Einheit angleichen; konservative Herstellergrenze |
| Lüftungsdauer | Eingabezahl | `input_number.klima_lueftungsdauer` | `input_number.ha_luftungsdauer` | Startwert 15 Minuten |
| Komfortzeit Wohnbereich | Zeitplan | `schedule.klima_wohnen_komfortzeit` | `schedule.ha_zeitplan_komfort_klima_wohnzimmer` | Zeitblöcke, in denen Komfort-Sollwert gelten soll |
| Komfortzeit Schlafzimmer | Zeitplan | `schedule.klima_schlafen_komfortzeit` | `schedule.ha_zeitplan_komfort_klima_schlafzimmer` | Tag-Komfort; außerhalb davon Nacht-/Eco-Sollwert |
| Komfortzeit Bad | Zeitplan | `schedule.klima_bad_komfortzeit` | `schedule.ha_zeitplan_komfort_klima_bad` | Komfortzeiten für das Bad |

Die Werte sind Vorschläge und müssen nach dem ersten Laden der YAML-Helfer eingestellt werden. Lege die drei Zeitpläne über **Einstellungen > Geräte & Dienste > Helfer > Helfer erstellen > Zeitplan** an und gib ihnen exakt die Entity-IDs `schedule.klima_wohnen_komfortzeit`, `schedule.klima_schlafen_komfortzeit` und `schedule.klima_bad_komfortzeit`. Stelle den Urlaubsbeginn auf die gewünschte lokale Uhrzeit und das Urlaubsende auf 23:59 des letzten Urlaubstags. So wird das Enddatum einschließlich behandelt.

## 1.3 Gruppen und abgeleitete Sensoren

Das YAML-Package legt die unten genannten Gruppen an. Wenn du stattdessen die Automationen einzeln in der UI erstellst, helfen diese Gruppen, Regeln nicht für jedes Fenster einzeln zu duplizieren:

- `binary_sensor.klima_fenster_wohnen_offen` --> `binary_sensor.ha_fenster_offen_wohnzimmer` : Ein, sobald eines der Wohnbereichsfenster offen ist.
- `binary_sensor.klima_fenster_schlafen_offen` --> `binary_sensor.ha_fenster_offen_schlafzimmer`: Ein, sobald eines der Schlafzimmerfenster offen ist.
- `binary_sensor.klima_regen` --> `binary_sensor.ha_regen`: Ein, sobald einer der relevanten Regensensoren Nässe meldet.
- `cover.klima_rolllaeden_west` --> `cover.ha_west_rollladen`: Gruppe der vier vorhandenen West-Rollläden. Ostfenster später separat gruppieren.

Gruppen nur für Entity-Typen verwenden, die Home Assistant als Gruppe unterstützt. Falls Fensterposition und Öffnungskontakt getrennte Entitäten sind, einen Template-Binärsensor für den kombinierten Zustand anlegen und die Einzelentitäten dokumentieren.

Für die Heizperiode kann ein Statistik-Sensor über die Wetter-Außentemperatur mit 24 Stunden Fenster und Charakteristik `mean` angelegt werden, z. B. `sensor.klima_aussentemperatur_24h_mittel`. Alternativ verwenden die Automationen einen anhaltenden Grenzwert mit `for: 48:00:00`; nach einem Home-Assistant-Neustart muss der Zustand erneut bewertet werden.

## 1.4 Vor Automationen einzeln prüfen

1. Die vier Thermostate je einzeln auf einen kleinen Sollwertwechsel testen und den vorherigen Sollwert wiederherstellen.
2. Daikin-Modi auflisten: `heat`, `cool`, `fan_only`, `off` müssen nur dann verwendet werden, wenn sie von der Entität tatsächlich unterstützt werden.
3. Jedes Dachfenster einzeln öffnen und schließen; beobachten, ob Position, Kontakt und Regensensor korrekt reagieren.
4. Jeden Rollladen einzeln bewegen und eine unterstützte Positionssteuerung feststellen.
5. Bei der Wetterintegration Vorhersage, Wind und Böen auf Aktualität prüfen. Bei unbekanntem oder veraltetem Wetter dürfen Fenster nicht automatisch öffnen.

Erst wenn diese Zuordnungen vollständig sind, die Vorschlags-IDs in den Automationen ersetzen.
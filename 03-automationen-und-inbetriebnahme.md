# 3. Automationen in Home Assistant einrichten

Die vollständige Package-Konfiguration liegt in [packages/klima_dachgeschoss.yaml](packages/klima_dachgeschoss.yaml). Sie enthält Gruppen, Template-/Statistik-Sensor, Skripte und Automationen. Die `replace_me`-IDs sind syntaktisch YAML, aber keine existierenden Home-Assistant-Entities; vor dem Laden müssen sie ersetzt werden. Die Einzelanleitungen weiter unten sind manuelle Alternativen. **Nicht gleichzeitig Package und dieselben Einzelautomationen aktivieren.**

## 3.1 Package installieren

1. Lege im Home-Assistant-Konfigurationsverzeichnis den Ordner `packages` an und kopiere das Package als `packages/klima_dachgeschoss.yaml` hinein.
2. Öffne `configuration.yaml`. Ergänze unter dem vorhandenen `homeassistant:`-Abschnitt:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

   Falls `homeassistant:` bereits existiert, nur `packages:` darunter ergänzen, keinen zweiten gleichnamigen Schlüssel anlegen.
3. Ersetze alle `replace_me`-Entity-IDs anhand der Zuordnungstabelle in Abschnitt 3.12.
4. Die drei Zeitplan-Entities aus [Schritt 1](01-entitaeten-und-helfer.md) müssen vor dem Neustart angelegt sein. Das Package erzeugt Modus-, Datum-, Zahlen-, Schalter- und Timer-Helfer sowie Gruppen, Statistik-Sensor, Template-Sensoren, Skripte und Automationen. Diese nicht zusätzlich in der UI anlegen. Bei Erstinstallation im Winter `input_boolean.klima_heizperiode` nach Prüfung der Außentemperatur einmal manuell einschalten; danach übernimmt die Hysterese.
5. Prüfe unter **Entwicklerwerkzeuge > YAML**, ob die Konfiguration gültig ist, und starte Home Assistant neu. Erst wenn alle Entities verfügbar sind, Automationen aktivieren.

Die Konfiguration wurde nicht gegen eine laufende Home-Assistant-Instanz getestet. Vor dem Laden Sicherung erstellen und, falls vorhanden, zunächst in einer Testinstanz prüfen.

## 3.2 Automationen erstellen und YAML-Modus öffnen

Für jede Automation:

1. Öffne **Einstellungen > Automatisierungen & Szenen > Automation erstellen > Neue Automation erstellen**.
2. Wähle im Menü oben rechts **Als YAML bearbeiten**. Falls du lieber visuell arbeitest, setze Trigger, Bedingungen und Aktionen nach den Tabellen unter der YAML-Vorlage.
3. Füge YAML ein, ersetze alle Platzhalter-Entity-IDs und speichere.
4. Öffne die Automation erneut und kontrolliere die Ablaufverfolgung (**Trace**) nach einem Testlauf.

YAML-Beispiele nutzen das aktuelle Automationsformat mit `triggers`, `conditions`, `actions` und `action:`. Home Assistant kann beim Speichern ältere Formate automatisch konvertieren.

## 3.3 Automation A: aktiven Raum-Sollwert setzen

Bei manueller Einzelanlage `input_boolean.klima_heizperiode` unter **Einstellungen > Geräte & Dienste > Helfer > Helfer erstellen > Umschalter** erstellen. Bei Package-Nutzung ist der Helfer bereits enthalten.

**Name:** `Klima - Sollwerte anwenden`

**Trigger im Editor:** Home Assistant Start sowie Änderungen an Anwesenheit, Urlaubssensor, Betriebsart, Timer, Zeitplänen, Sollwert-Helfern und Heizperioden-Umschalter.

**Logik:** Urlaub zuerst, danach Abwesenheit, dann aktive manuelle Übersteuerung, zuletzt Zuhause-Zeitplan. Die Heizperiode wählt zwischen Komfort-/Eco-Sollwert und niedrigem Schutz-Sollwert. Der folgende YAML-Block zeigt das Muster für die drei Wohnbereichs-DECT-301. Dasselbe `choose`-Muster für Bad und später Schlafzimmer mit ihren eigenen Helfern wiederholen.

```yaml
alias: Klima - Sollwerte Wohnbereich
id: klima_sollwerte_wohnbereich
mode: restart
triggers:
  - trigger: homeassistant
    event: start
  - trigger: state
    entity_id:
      - binary_sensor.klima_urlaub_aktiv
      - group.haushalt
      - input_select.klima_betriebsart
      - timer.klima_manuelle_uebersteuerung
      - schedule.klima_wohnen_komfortzeit
      - input_number.klima_wohnen_komfort
      - input_number.klima_wohnen_eco
      - input_number.klima_schutz_sollwert
      - input_number.klima_urlaub_sollwert
      - input_boolean.klima_heizperiode
actions:
  - choose:
      - conditions:
          - condition: state
            entity_id: binary_sensor.klima_urlaub_aktiv
            state: "on"
        sequence:
          - action: climate.set_temperature
            target:
              entity_id:
                - <ID_DECT301_WOHNEN_1>
                - <ID_DECT301_WOHNEN_2>
                - <ID_DECT301_WOHNEN_3>
            data:
              temperature: "{{ states('input_number.klima_urlaub_sollwert') | float(16) }}"
      - conditions:
          - condition: state
            entity_id: group.haushalt
            state: not_home
        sequence:
          - action: climate.set_temperature
            target:
              entity_id:
                - <ID_DECT301_WOHNEN_1>
                - <ID_DECT301_WOHNEN_2>
                - <ID_DECT301_WOHNEN_3>
            data:
              temperature: "{{ states('input_number.klima_wohnen_eco') | float(17) }}"
      - conditions:
          - condition: state
            entity_id: input_select.klima_betriebsart
            state: Manuell
        sequence: []
    default:
      - choose:
          - conditions:
              - condition: state
                entity_id: input_boolean.klima_heizperiode
                state: "on"
              - condition: state
                entity_id: schedule.klima_wohnen_komfortzeit
                state: "on"
            sequence:
              - action: climate.set_temperature
                target:
                  entity_id:
                    - <ID_DECT301_WOHNEN_1>
                    - <ID_DECT301_WOHNEN_2>
                    - <ID_DECT301_WOHNEN_3>
                data:
                  temperature: "{{ states('input_number.klima_wohnen_komfort') | float(20) }}"
          - conditions:
              - condition: state
                entity_id: input_boolean.klima_heizperiode
                state: "on"
            sequence:
              - action: climate.set_temperature
                target:
                  entity_id:
                    - <ID_DECT301_WOHNEN_1>
                    - <ID_DECT301_WOHNEN_2>
                    - <ID_DECT301_WOHNEN_3>
                data:
                  temperature: "{{ states('input_number.klima_wohnen_eco') | float(17) }}"
        default:
          - action: climate.set_temperature
            target:
              entity_id:
                - <ID_DECT301_WOHNEN_1>
                - <ID_DECT301_WOHNEN_2>
                - <ID_DECT301_WOHNEN_3>
            data:
              temperature: "{{ states('input_number.klima_schutz_sollwert') | float(12) }}"
```

**Wichtig:** Das Package definiert `group.haushalt` aus den Person-Entities. Falls du nur eine Person hast, kannst du in der Automation stattdessen direkt `person.<name>` verwenden. Gruppenstatus ist `home`, sobald mindestens eine Person zuhause ist. Die leere Sequenz im Manuell-Zweig bedeutet: während des Timers keinen Sollwert überschreiben.

Erstelle eine analoge Bad-Automation mit `input_number.klima_bad_komfort`, `input_number.klima_bad_eco` und `input_number.klima_schutz_sollwert`. Schlafzimmer erst nach Installation des DECT 301 ergänzen; dort Komfortzeitplan an Tag-/Nacht-Helfer koppeln. Außerhalb der Heizperiode wird der Schutz-Sollwert gesetzt, damit normale Heizanforderungen weitgehend unterdrückt werden. Falls das DECT-301-Entity einen HVAC-Modus `off` unterstützt, kann stattdessen eine echte Abschaltung getestet werden; den Schutz gegen Frost im Gebäude nicht allein Home Assistant überlassen.

## 3.4 Automation B: manuelle Übersteuerung zurücksetzen

**Name:** `Klima - manuelle Übersteuerung befristen`

- Trigger: `input_select.klima_betriebsart` wechselt auf `Manuell`.
- Aktion: `timer.start` für `timer.klima_manuelle_uebersteuerung`, Dauer `02:00:00`.
- Zweite Automation: Trigger Ereignis `timer.finished`, Bedingung Timer-Entity ist `timer.klima_manuelle_uebersteuerung`, Aktion `input_select.select_option` mit Option `Automatik`.

Wenn Urlaub beginnt oder alle Personen das Haus verlassen, zusätzlich den Timer abbrechen und Betriebsart auf `Automatik` setzen. Urlaub/Abwesenheit haben in der Sollwertautomation unabhängig vom Modus Vorrang.

## 3.5 Automationen C/D: Heizperiode mit Hysterese

Bei Package-Nutzung sind Statistik-Sensor und Automationen enthalten. Für manuelle Einzelanlage den Statistik-Helfer wie in Schritt 2 beschrieben erstellen.

**C `Klima - Heizperiode einschalten`:** Trigger numerischer Zustand unter `input_number.klima_heizperiode_ein` für `48:00:00`; Bedingung Heizperiode ist aus; Aktion `input_boolean.turn_on` auf `input_boolean.klima_heizperiode`.

**D `Klima - Heizperiode ausschalten`:** Trigger numerischer Zustand über `input_number.klima_heizperiode_aus` für `48:00:00`; Bedingung Heizperiode ist an; Aktion `input_boolean.turn_off`.

YAML-Form des Einschalt-Triggers:

```yaml
triggers:
  - trigger: numeric_state
    entity_id: <ID_AUSSENTEMPERATUR_24H_MITTEL>
    below: input_number.klima_heizperiode_ein
    for: "48:00:00"
```

Zum Ausschalten `below` durch `above` und den Helfer durch `input_number.klima_heizperiode_aus` ersetzen. Beim Home-Assistant-Start Werte prüfen; ohne vollständige 24-Stunden-Historie nicht automatisch in den Sommerzustand wechseln. Heizperiode ist eine Sollwertfreigabe, kein direkter Aufruf zum Heizen.

## 3.6 Automation E: Fenster bei Regen/Windrisiko schließen

**Name:** `Klima - Fenster bei Risiko schließen`

Erstelle Trigger für jeden Regensensor (`on`) und für einen optionalen Template-Sensor, der Wind-/Böenüberschreitung oder ungültige Sicherheitsdaten zusammenfasst. Bedingungen sind nicht nötig: Schließen ist im Urlaub und bei Abwesenheit erlaubt. Aktion `cover.close_cover` für ausschließlich die Dachfenster-Entities ausführen, nicht für Rollläden. Falls die Fensterintegration keinen `cover`-Typ nutzt, verwende ihren tatsächlich unterstützten Schließdienst.

Zunächst nur mit einem Fenster testen. Die Regensensoren einzeln nass testen bzw. den Zustand im HA-Entwicklerwerkzeug simulieren und Trace prüfen. Automatisches Öffnen deaktiviert lassen, wenn Wind/Böen fehlen oder die Zuverlässigkeit der lokalen Vorhersage nicht belegt ist.

## 3.7 Automation F: natürlich lüften

Lege pro Raum eine Automation an. Öffnen ist der riskanteste Teil; erst nach erfolgreichem Test des Sicherheitsschließens aktivieren.

**Trigger:** Raumtemperatur steigt über den Lüftungsschwellenwert und Wettervorhersage wird aktualisiert; zusätzlich stündliche Prüfung.

**Bedingungen vor Öffnen:** Urlaubssensor `off`; Raumfenster geschlossen; Innenraum über aktivem Sollwert; Außentemperatur mindestens 2 °C niedriger; Wetter-Entity und Wind-/Böenwerte gültig und aktuell; kein Regen; Wind unter dem vom Velux-Hersteller zulässigen Limit. Bei normaler Abwesenheit darf geöffnet werden, aber nur für einen begrenzten Timer (Start 15 Minuten).

**Aktionen:** Daikin ausschalten; ein einzelnes Fenster im Raum öffnen; Timer starten. Separate Schließautomation schließt bei Timer-Ende, Regen, Windüberschreitung, fehlenden Wetterdaten, Urlaubsbeginn oder wenn außen nicht mehr ausreichend kühler ist. Das Schließen darf nie durch Urlaub blockiert werden.

Die Wettervorhersage in HA testen: **Entwicklerwerkzeuge > Aktionen > `weather.get_forecasts`**, Wetter-Entity auswählen, Typ `hourly`, Antwortvariable setzen und Aktion ausführen. Vor der Template-Bedingung die tatsächliche Antwort der Integration untersuchen. Windfelder sind nicht bei jedem Anbieter gleich benannt; keine angenommene Feldbezeichnung ungeprüft in die Automatik übernehmen. Wenn Böen nicht in der Vorhersage enthalten sind, Fenster nicht automatisiert öffnen.

## 3.8 Automation G: West-Rollläden vorausschauend beschatten

**Name:** `Klima - Westbeschattung`

1. Gruppe `cover.klima_rolllaeden_west` aus den vier West-Rollläden erstellen.
2. Trigger: stündlich sowie bei deutlicher Temperaturänderung des AVM 440 und zu einem frühen Nachmittagszeitpunkt.
3. Bedingungen: Fenstergruppe West bekannt; Raum-/Wetterwerte valide; Sonne für Westfassade relevant. Beschattungsbeginn nach Test zunächst konservativ festlegen.
4. Aktion: `cover.close_cover` auf Westgruppe, wenn warmer/s sonniger Nachmittag prognostiziert ist oder die Raumtemperatur steigt. Im Winter bei Sonneneinstrahlung und Heizbedarf öffnen.
5. Urlaub blockiert Beschattung nicht. Fensteröffnungsregeln sind davon unabhängig.

Die Forecast-Antwort aus `weather.get_forecasts` ist integrationsabhängig. Wenn hourly forecast kein Einstrahlungsfeld liefert, erst mit Sonnenstand und realer Innentemperatur arbeiten; keine nicht vorhandenen Attribute referenzieren. Ostfenster später als eigene Gruppe und mit Morgenlogik ergänzen.

## 3.9 Automation H: Daikin-Kühlung im Sommer

Nur einrichten, wenn `cool` in den unterstützten Daikin-Modi steht. Inbetriebnahme zunächst ausschließlich bei Abwesenheit.

**Einschalten**, wenn alle Bedingungen gelten: AVM-440-Wohnraumtemperatur >= `input_number.klima_kuehlung_ein` (Start 25,5 °C), alle Fenster geschlossen, Sensoren verfügbar, Daikin verfügbar, keine Urlaubsregel für aktive Komfortkühlung. Aktion zuerst `climate.set_hvac_mode` mit `cool`, danach `climate.set_temperature` z. B. auf 24,5 °C. Ziel nach Messvergleich kalibrieren.

**Ausschalten** bei AVM-440-Temperatur <= `input_number.klima_kuehlung_aus` (Start 24,5 °C), Fensteröffnung, Urlaub oder nicht verfügbarem Sensor. Fenster-/Sensor-Sicherheitsabschaltung sofort; normale Temperaturabschaltung mit Mindestlaufzeit. Bei Anwesenheit nur innerhalb des Komfortzeitplans und mit `input_number.klima_wohnen_kuehlziel` kühlen.

```yaml
alias: Klima - Abwesenheitskühlung starten
id: klima_abwesenheitskuehlung_start
mode: single
triggers:
  - trigger: template
    value_template: >-
      {{ is_number(states('<ID_AVM440_TEMPERATUR>'))
         and states('<ID_AVM440_TEMPERATUR>') | float(0)
             >= states('input_number.klima_kuehlung_ein') | float(25.5) }}
conditions:
  - condition: state
    entity_id: group.haushalt
    state: not_home
  - condition: state
    entity_id: binary_sensor.klima_urlaub_aktiv
    state: "off"
  - condition: state
    entity_id: <ID_WOHNBEREICH_FENSTERGRUPPE>
    state: "off"
  - condition: template
    value_template: "{{ 'cool' in state_attr('<ID_DAIKIN>', 'hvac_modes') }}"
actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: <ID_DAIKIN>
    data:
      hvac_mode: cool
  - action: climate.set_temperature
    target:
      entity_id: <ID_DAIKIN>
    data:
      temperature: 24.5
```

Erstelle eine zweite Stop-Automation mit Template-Trigger unter der Ausschaltgrenze und zusätzlichen State-Triggern für Fensteröffnung, Urlaub und Sensor `unavailable`. Für Sensor-/Fenster-Trigger Daikin sofort auf `off`; beim Temperaturtrigger nur ausschalten, wenn sie gerade kühlt. Die Template-Auslöser müssen für den echten Sensor getestet werden, bevor Kühlung unbeaufsichtigt läuft.

Die Grenze 26 °C ist ein Ziel, keine garantierte Maximaltemperatur. Bei extremer Außenhitze oder fehlender Kühlleistung kann sie überschritten werden.

## 3.10 Automation I: Daikin-Heiz-Boost

**Name:** `Klima - Daikin Heiz-Boost`

**Trigger:** AVM-440-Temperatur fällt mindestens 1,5 °C unter den effektiven Wohnbereichs-Sollwert und bleibt dort 15 Minuten. Die Abweichung als Template-Trigger abbilden; Temperaturfühler muss numerisch sein.

**Bedingungen:** Zuhause, Heizperiode an, Wohnbereich-Komfortzeitplan an, alle Fenster geschlossen, Urlaub aus, Daikin verfügbar, `heat` in `hvac_modes`. Zusätzlich `input_select.klima_betriebsart` nicht `Manuell` oder dessen Timer inaktiv.

**Aktionen:** HVAC-Modus `heat`; Solltemperatur auf den Komfortwert (oder maximal Komfortwert + 1 °C als kurzzeitiger Boost) setzen. Stop-Automation bei erreichtem Sollwert mit Hysterese, Fensteröffnung, Abwesenheit/Urlaub, Heizperiodenende oder maximaler Laufzeit. Heizkörper zunächst auf normalem Sollwert belassen.

Erst im Trace testen. Eine eigenständige Boost-Automation darf nicht zeitgleich mit Kühlung oder Ventilator-Modus Daikin-Befehle senden; Bedingungen müssen diese Modi gegenseitig ausschließen.

## 3.11 Automation J: warme Luft mit Daikin-Ventilator durchmischen

Nur verwenden, wenn `fan_only` unterstützt wird. **Trigger:** Template bleibt 10 Minuten wahr, wenn Daikin-Temperatur minus AVM-440-Temperatur >= 2 °C. **Bedingungen:** Heizperiode an; Zuhause; Fenster geschlossen; Sensoren numerisch; Daikin-Modus `heat`; `hvac_action` ist `idle`; Kühlung/Boost nicht aktiv.

**Aktionen:** Daikin-Sollwert zunächst mit Variable sichern, HVAC-Modus `fan_only`, 10 Minuten warten, dann Modus `heat` und gesicherten Sollwert zurücksetzen. `mode: single`, damit keine zweite Sequenz parallel startet; anschließend 30 Minuten Abkühlpause. Zusätzlich Start-/Recovery-Automation anlegen, die bei Home-Assistant-Start prüft, ob Daikin unerwartet auf `fan_only` steht und gegebenenfalls auf `heat` zurücksetzt. Erst Werte protokollieren, dann aktivieren.

Vor dem Speichern prüfen, dass die Daikin-Integration `fan_only` tatsächlich unterstützt und dass `hvac_action` den Status zuverlässig aktualisiert. Ist eine Sensorentität `unknown`/`unavailable`, darf diese Automation nicht starten.

## 3.12 Entity-IDs einsetzen

Ersetze die Mapping-IDs in `packages/klima_dachgeschoss.yaml`:

| Package-ID | Benötigte echte Entity |
|---|---|
| `person.replace_me_person_1` | HA-Person; weitere Personen in der Gruppe `haushalt` ergänzen |
| `binary_sensor.replace_me_fenster_wohnen_1..4` | Wohnbereich-Fensterkontakte, `on` = offen |
| `binary_sensor.replace_me_fenster_schlafen_1..2` | Schlafzimmer-Fensterkontakte, `on` = offen |
| `binary_sensor.replace_me_regen_wohnen_1..4` | Regensensoren im Wohnbereich, `on` = Regen |
| `binary_sensor.replace_me_regen_schlafen_1..2` | Regensensoren im Schlafzimmer |
| `sensor.replace_me_wetter_aussentemperatur` | Außentemperatur der Wetterintegration |
| `sensor.klima_windgeschwindigkeit` | wird aus `wind_speed` der Wetter-Entity erzeugt; Einheit muss zum Helfer `input_number.klima_max_windgeschwindigkeit` passen |
| `sensor.replace_me_avm440_temperatur` | AVM-440-Temperatur auf Aufenthaltsniveau |
| `sensor.replace_me_daikin_temperatur` | Daikin-Innentemperatur zur Schichtungserkennung |
| `weather.replace_me_home` | Wetter-Entity mit aktuellem `wind_speed` sowie stündlichen Forecast-Feldern `temperature` und `wind_speed` |
| `climate.replace_me_dect301_wohnen_1..3` | drei Wohnbereichs-DECT-301 |
| `climate.replace_me_dect301_bad` | Bad-DECT-301 |
| `climate.replace_me_daikin` | Daikin-Climate-Entity |
| `cover.replace_me_fenster_wohnen_1..4` und `cover.replace_me_fenster_schlafen_1..2` | Velux-Fensterantriebe für Öffnen/Schließen |
| `cover.replace_me_rollladen_wohnen_1..4` | West-Rollläden |

Die echten Domains können abweichen. Wenn ein Fenster kein `cover`-Entity ist, `cover.open_cover`/`cover.close_cover` durch den Integrationsdienst ersetzen. Die Lüftungsautomation ist standardmäßig deaktiviert. Forecast-Windgeschwindigkeit ist nicht dasselbe wie lokale Böenmessung; fehlt eines der Forecast-Felder, bleibt die Öffnungsaktion gesperrt.

## 3.13 Reihenfolge der Inbetriebnahme

1. **Beobachten:** Entitäten und Sensorabweichungen einige Tage protokollieren; keine Aktor-Automation.
2. **Sollwerte:** Automation A zunächst nur für einen Wohnbereichs-DECT-301 aktivieren; Komfort, Eco, Urlaub und Rückkehr in Automatik prüfen.
3. **Heizperiode:** Statistik-Helfer auf Historie prüfen, dann C/D aktivieren.
4. **Beschattung:** Einen West-Rollladen testen, Bewegungsrichtung und Status verifizieren, dann Gruppe.
5. **Daikin:** Modi und Sollwertsteuerung manuell testen; anschließend Kühlung, Heiz-Boost und Ventilator einzeln aktivieren.
6. **Fenster schließen:** Regen- und Sicherheitsschließen testen.
7. **Fenster öffnen:** Nur bei verifizierten Wind-/Böenfeldern und Herstellergrenze aktivieren. Zunächst Benachrichtigung statt Öffnen.
8. **Urlaub:** kurzen Testzeitraum setzen. Prüfen: kein Öffnen, richtiger Sollwert, Beschattung bleibt möglich, Sicherheits-Schließen bleibt aktiv.
9. **Fehlerfälle:** Temperatur/Wetter kurzzeitig `unavailable` simulieren. Erwartung: keine Öffnung und kein unkontrollierter Daikin-Start; Trace und Benachrichtigung prüfen.

Nach jedem Schritt in der Automation **Trace** öffnen. Nicht mehrere Daikin-Automationen gleichzeitig aktivieren, bevor gegenseitige Sperren getestet sind. Keine Konfiguration mit unbekannten Entity-IDs speichern.
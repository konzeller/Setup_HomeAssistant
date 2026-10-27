# Dachgeschosswohnung: intelligente automatisierte Klimaregelung

# Umgebung
RaspberryPi mit Home Assistant

# Zimmer und Geräte
## Badezimmer
- 1x AVM Dect 301

## Schlafzimmer
- 2x Velux Solar Dachfenster
- 2x Velux Solar Rolladen
- 1x Velux Raumklimasensor

## Wohn-/Esszimmer und Küche
- 4x Velux Solar Dachfenster
- 4x Velux Solar Rolladen
- 1x AVM Dect 440
- 3x AVM Dect 301
- 1x Daikin Stylish 5KW Wärmepumpe
- Phillips Hue Leuchten und Bridge

# Umsetzungsanleitung

Die Home-Assistant-Regelung ist in einzelne Schritte aufgeteilt:

1. [Entitäten erfassen und Helfer anlegen](01-entitaeten-und-helfer.md)
2. [Betriebsmodi und Regelungslogik](02-betriebsmodi-und-logik.md)
3. [Automationen einrichten und testen](03-automationen-und-inbetriebnahme.md)
4. [Home-Assistant-YAML-Package](packages/klima_dachgeschoss.yaml)

Die Entity-IDs in der Anleitung sind Vorschläge beziehungsweise Platzhalter.
Vor dem Einrichten müssen sie durch die tatsächlichen IDs aus **Entwicklerwerkzeuge > Zustände** ersetzt werden. Die Geräteübersicht oben enthält bisher keine IDs.
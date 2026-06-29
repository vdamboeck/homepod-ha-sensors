# Home Assistant Setup

## Voraussetzungen

- Home Assistant 2024.8 oder neuer
- [HomeKit Bridge](https://www.home-assistant.io/integrations/homekit/) Integration aktiv

---

## Schritt 1: Helper anlegen

Öffne **Einstellungen → Geräte & Dienste → Helper → Helper erstellen**.

### 1a. Boolean Helper (einmalig)

| Feld | Wert |
|------|------|
| Typ | Schalter |
| Name | `HomePod Trigger` (oder beliebig) |
| Icon | `mdi:toggle-switch` |

Notiere dir die Entity-ID (z. B. `input_boolean.homepod_trigger`) — du brauchst sie später.

### 1b. Numerische Helper (pro HomePod, 2×)

Für jeden HomePod zwei Helfer anlegen:

**Temperatur:**

| Feld | Wert |
|------|------|
| Typ | Zahl |
| Name | `HomePod 1 Temperatur` |
| Icon | `mdi:thermometer` |
| Min | `-20` |
| Max | `60` |
| Schrittweite | `0.1` |
| Einheit | `°C` |
| Anzeigemodus | Eingabefeld |

**Luftfeuchtigkeit:**

| Feld | Wert |
|------|------|
| Typ | Zahl |
| Name | `HomePod 1 Luftfeuchtigkeit` |
| Icon | `mdi:water-percent` |
| Min | `0` |
| Max | `100` |
| Schrittweite | `0.1` |
| Einheit | `%` |
| Anzeigemodus | Eingabefeld |

Für jeden weiteren HomePod diesen Schritt wiederholen und den Namen anpassen (`HomePod 2 Temperatur` usw.).

Die YAML-Referenz für alle Helper findest du in [`ha_config/helpers.yaml`](../ha_config/helpers.yaml).

### 1c. Datetime-Helper (optional, pro HomePod)

Falls du auf dem Dashboard sehen möchtest, wann die letzte Messung eingegangen ist:

| Feld | Wert |
|------|------|
| Typ | Datum und/oder Uhrzeit |
| Name | `HomePod 1 Letzte Aktualisierung` |
| Datum einschließen | Ja |
| Uhrzeit einschließen | Ja |

---

## Schritt 2: Template-Sensoren anlegen

Template-Sensoren sorgen für korrekte Einheiten, Geräte-Klassen und Langzeitstatistiken.

**Für jeden HomePod drei Template-Sensoren anlegen** (Temperatur, Luftfeuchtigkeit, Absolute Luftfeuchtigkeit):

Öffne **Einstellungen → Geräte & Dienste → Helper → Helper erstellen → Template → Sensor-Template**.

Beispiel für HomePod 1 Temperatur:

| Feld | Wert |
|------|------|
| Name | `HomePod 1 Temperatur` |
| Zustandstemplate | `{{ states('input_number.homepod_1_temperatur') \| float(0) }}` |
| Geräteklasse | `Temperatur` |
| Zustandsklasse | `Messung` |
| Maßeinheit | `°C` |

Die vollständigen Templates (inkl. Absolute Luftfeuchtigkeit via Magnus-Formel) findest du in [`ha_config/template_sensors.yaml`](../ha_config/template_sensors.yaml).

> **Hinweis:** Passe die Entity-IDs der `input_number`-Helper im Template an deine eigenen an. Die Entity-ID findest du unter **Entwicklerwerkzeuge → Zustände**.

---

## Schritt 3: Trigger-Blueprint importieren

[![Import Trigger Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?url=https%3A%2F%2Fgithub.com%2Fvdamboeck%2Fhomepod-ha-sensors%2Fblob%2Fmain%2Fblueprints%2Fhomepod_trigger.yaml)

Nach dem Import:

1. Öffne **Einstellungen → Automationen → Blueprint-Automation erstellen → HomePod Sensor – Trigger**
2. Konfiguriere:
   - **Boolean Helper:** wähle den in Schritt 1a angelegten Helper (`input_boolean.homepod_trigger`)
   - **Polling-Intervall:** Standard 5 Minuten empfohlen — Apple Home drosselt häufigere Automationsläufe, das Minimum liegt bei ca. 2–3 Minuten; außerdem aktualisieren HomePod-Sensoren intern nicht sekundengenau, sodass kürzere Intervalle keinen Mehrwert bringen (siehe [`01_setup_overview.md`](01_setup_overview.md))
   - **Einschaltdauer:** Standard 3 Sekunden empfohlen
3. Speichern

---

## Schritt 4: Webhook-Blueprint importieren

[![Import Webhook Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?url=https%3A%2F%2Fgithub.com%2Fvdamboeck%2Fhomepod-ha-sensors%2Fblob%2Fmain%2Fblueprints%2Fhomepod_webhook_receiver.yaml)

Nach dem Import:

1. Öffne **Einstellungen → Automationen → Blueprint-Automation erstellen → HomePod Sensor – Webhook Empfänger**
2. Konfiguriere die **Webhook-ID** (Standard: `homepod_daten` — merke dir diesen Wert, du brauchst ihn in Apple Home)
3. Konfiguriere **HomePod 1 (erforderlich)**:
   - **Payload-Schlüssel Temperatur:** z. B. `homepod1_temperature` — dieser Key muss exakt mit dem Kurzbefehl in Apple Home übereinstimmen
   - **Numerischer Helper – Temperatur:** wähle den in Schritt 1b angelegten Helper
   - **Payload-Schlüssel Luftfeuchtigkeit:** z. B. `homepod1_humidity`
   - **Numerischer Helper – Luftfeuchtigkeit:** entsprechend
4. Für jeden weiteren HomePod einen der optionalen Slots (HomePod 2–5) aufklappen und identisch konfigurieren
5. Speichern

> **Tipp:** Die Payload-Schlüssel sind frei wählbar, müssen aber exakt mit den Keys übereinstimmen, die der Kurzbefehl in Apple Home sendet (Groß-/Kleinschreibung beachten).

---

## Schritt 5: HomeKit Bridge prüfen

1. Öffne **Einstellungen → Geräte & Dienste → HomeKit Bridge → Entitäten**
2. Suche nach dem Boolean Helper (`input_boolean.homepod_trigger`)
3. Falls er nicht auftaucht: HomeKit Bridge → Einstellungen → Filter anpassen und `input_boolean` einschließen
4. Nach Änderungen HomeKit Bridge neu starten und ggf. den QR-Code in Apple Home neu koppeln

Der Helper erscheint in Apple Home als Schalter und kann dort als Auslöser für eine Automation verwendet werden.

---

## Verifikation

Teste die gesamte Kette, bevor du Apple Home konfigurierst:

1. Öffne **Entwicklerwerkzeuge → Zustände**
2. Setze `input_boolean.homepod_trigger` manuell auf `on`
3. Sende manuell einen Test-Webhook:

```bash
curl -X POST https://YOUR_HA_URL/api/webhook/homepod_daten \
  -H "Content-Type: application/json" \
  -d '{"homepod1_temperature": "21.5", "homepod1_humidity": "55.0"}'
```

4. Prüfe, ob `input_number.homepod_1_temperatur` den Wert `21.5` zeigt
5. Prüfe, ob `sensor.homepod_1_temperatur` ebenfalls `21.5 °C` zeigt

Wenn das funktioniert, ist die HA-Seite vollständig eingerichtet. Weiter mit [`03_apple_home_setup.md`](03_apple_home_setup.md).

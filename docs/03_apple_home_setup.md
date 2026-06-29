# Apple Home Setup

Dieser Teil des Setups ist vollständig manuell — Apple stellt keine API bereit, über die HA direkt auf HomePod-Sensordaten zugreifen könnte. Du konfigurierst eine Apple-Home-Automation, die einen Kurzbefehl ausführt. Der Kurzbefehl liest die Sensordaten aller HomePods aus und sendet sie als JSON an den HA-Webhook.

## Voraussetzungen

- Mindestens ein HomePod mini als Home Hub aktiv
- Der Boolean Helper aus Schritt 1a der HA-Einrichtung ist in Apple Home als Schalter sichtbar
- Du kennst die Webhook-URL deiner HA-Instanz: `http://DEINE_HA_IP:8123/api/webhook/homepod_daten`

> **Hinweis zur URL:** Die Webhook-Automation ist auf `local_only: true` gesetzt — sie akzeptiert nur Anfragen aus dem lokalen Netzwerk. Verwende daher die lokale IP-Adresse deiner HA-Instanz, nicht eine externe URL.

---

## Teil A: Kurzbefehl erstellen

Der Kurzbefehl wird von Apple Home ausgeführt, wenn der Boolean Helper eingeschaltet wird. Er liest alle HomePods aus und sendet die Daten in einem einzigen HTTP-POST an HA.

### Schritt-für-Schritt

Öffne die **Kurzbefehle-App** auf iPhone oder iPad.

#### 1. Neuen Kurzbefehl erstellen

Tippe auf **+** (oben rechts) und benenne ihn z. B. „HomePod Sensoren".

#### 2. Sensordaten abrufen — HomePod 1

Füge für jeden HomePod zwei Aktionen hinzu:

**Aktion:** „Heimzubehör steuern"
- Wähle deinen HomePod 1 aus der Raumliste
- Wähle **Temperatur abrufen**
- Speichere das Ergebnis in einer Variable, z. B. `hp1_temp`

**Aktion:** „Heimzubehör steuern"
- Wähle denselben HomePod 1
- Wähle **Relative Luftfeuchtigkeit abrufen**
- Speichere das Ergebnis in einer Variable, z. B. `hp1_hum`

*Wiederhole diesen Schritt für jeden weiteren HomePod mit entsprechenden Variablennamen (`hp2_temp`, `hp2_hum` usw.).*

#### 3. Absicherung gegen Ausfall (empfohlen)

Füge eine **Wenn-Bedingung** ein:
- **Wenn** `hp1_temp` **ist** leer → **Stopp**

Das verhindert, dass bei einem kurzzeitig offline gegangenen HomePod der Wert `0` als gültige Messung in HA geschrieben wird.

#### 4. HTTP-POST senden

**Aktion:** „Inhalt aus URL abrufen"

| Feld | Wert |
|------|------|
| URL | `http://DEINE_HA_IP:8123/api/webhook/homepod_daten` |
| Methode | POST |
| Anfragetext | JSON |

Füge im JSON-Body die Schlüssel-Wert-Paare ein. Die Schlüsselnamen müssen **exakt** mit den Payload-Schlüsseln übereinstimmen, die du im Webhook-Blueprint konfiguriert hast:

**Beispiel für einen einzelnen HomePod:**

```json
{
  "homepod1_temperature": "<Variable hp1_temp>",
  "homepod1_humidity":    "<Variable hp1_hum>"
}
```

**Beispiel für zwei HomePods:**

```json
{
  "homepod1_temperature": "<Variable hp1_temp>",
  "homepod1_humidity":    "<Variable hp1_hum>",
  "homepod2_temperature": "<Variable hp2_temp>",
  "homepod2_humidity":    "<Variable hp2_hum>"
}
```

> **Hinweis zu Dezimaltrennern:** Apple Shortcuts liefert Messwerte je nach iOS-Version und Gerätesprache mit Komma oder Punkt als Dezimaltrenner (z. B. `21,5` statt `21.5`). Die Webhook-Automation in HA normalisiert das automatisch — du musst hier nichts weiter tun.

#### 5. Kurzbefehl testen

Tippe auf **Ausführen** (▶) und prüfe anschließend in HA unter **Entwicklerwerkzeuge → Zustände**, ob die `input_number`-Helper die erwarteten Werte zeigen.

---

## Teil B: Apple-Home-Automation erstellen

Öffne die **Home-App** auf iPhone oder iPad.

### Schritt-für-Schritt

#### 1. Neue Automation anlegen

Tippe auf **+** im Tab „Automation" und wähle **Ein Zubehörstatus ändert sich**.

#### 2. Auslöser konfigurieren

- Wähle den Boolean Helper aus HA (erscheint als Schalter, z. B. „HomePod Trigger")
- **Auslöser:** Wenn eingeschaltet wird

#### 3. Aktion hinzufügen

- Tippe auf **Kurzbefehl ausführen**
- Wähle den in Teil A erstellten Kurzbefehl „HomePod Sensoren"

#### 4. Automation aktivieren

Speichern und sicherstellen, dass die Automation aktiviert ist.

---

## Platzhalter für Screenshots

*Füge hier Screenshots der einzelnen Kurzbefehl-Schritte und der Apple-Home-Automation ein.*

| Dateiname | Inhalt |
|-----------|--------|
| `screenshots/shortcut_overview.png` | Gesamtansicht des Kurzbefehls |
| `screenshots/shortcut_get_sensor.png` | Aktion „Heimzubehör steuern" |
| `screenshots/shortcut_http_post.png` | Aktion „Inhalt aus URL abrufen" mit JSON-Body |
| `screenshots/apple_home_automation.png` | Apple-Home-Automation mit Auslöser und Aktion |

---

## Gesamttest

Wenn alles konfiguriert ist:

1. Öffne **Home Assistant → Entwicklerwerkzeuge → Zustände**
2. Setze `input_boolean.homepod_trigger` manuell auf `on`
3. Warte ca. 5–10 Sekunden
4. Prüfe:
   - `input_number.homepod_1_temperatur` — zeigt einen plausiblen Wert (nicht `0`)
   - `sensor.homepod_1_temperatur` — zeigt denselben Wert in °C
   - `sensor.homepod_1_luftfeuchtigkeit` — zeigt einen plausiblen Wert in %
   - Falls angelegt: `input_datetime.homepod_1_letzte_aktualisierung` — zeigt den aktuellen Zeitstempel

Ab diesem Punkt läuft das Polling automatisch alle N Minuten.

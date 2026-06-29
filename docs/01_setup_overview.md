# Setup-Übersicht

## Das Problem

Die Apple TV Integration in Home Assistant leitet HomePod-Sensordaten (Temperatur, Luftfeuchtigkeit) **nicht** an HA weiter. HomePods exponieren diese Daten nur innerhalb des Apple-Home-Ökosystems — es gibt keinen offiziellen, direkten Weg in Home Assistant.

## Der Umweg

Dieser Workflow überbrückt die Lücke über drei Stufen:

1. **HA triggert Apple Home** — über einen Boolean Helper, der via HomeKit Bridge in Apple Home sichtbar ist
2. **Apple Home liest die Sensoren** — eine Automation startet einen Kurzbefehl, der alle HomePods abfragt
3. **Apple Home schickt die Daten zurück** — als HTTP-POST an einen HA-Webhook, ausschließlich im LAN

## Datenfluss

```
┌──────────────────────────────────────────────────────────────┐
│                      Home Assistant                          │
│                                                              │
│   Zeitgeber (alle N Minuten)                                 │
│     └── input_boolean [TRIGGER] EIN  ──┐                    │
│         (nach 3 Sek.) AUS  ◄───────────┘                    │
│                  │                                           │
│            HomeKit Bridge                                    │
└──────────────────┼───────────────────────────────────────────┘
                   │  (HomeKit-Protokoll, LAN)
┌──────────────────▼───────────────────────────────────────────┐
│                       Apple Home                             │
│                                                              │
│   Automation: wenn TRIGGER eingeschaltet wird                │
│     └── Kurzbefehl ausführen                                 │
│           ├── HomePod 1: Temperatur abrufen                  │
│           ├── HomePod 1: Luftfeuchtigkeit abrufen            │
│           ├── HomePod 2: Temperatur abrufen  (optional)      │
│           ├── HomePod 2: Luftfeuchtigkeit abrufen  (opt.)    │
│           └── HTTP-POST an HA-Webhook (alle Werte auf einmal)│
└──────────────────────────────────────────────────────────────┘
                   │  (HTTP POST, LAN only)
┌──────────────────▼───────────────────────────────────────────┐
│                      Home Assistant                          │
│                                                              │
│   Webhook-Automation                                         │
│     ├── input_number.homepod_1_temperatur      → Wert        │
│     ├── input_number.homepod_1_luftfeuchtigkeit → Wert       │
│     ├── input_number.homepod_2_temperatur      → Wert (opt.) │
│     └── input_number.homepod_2_luftfeuchtigkeit → Wert (opt.)│
│                  │                                           │
│   Template-Sensor (liest input_number)                       │
│     ├── sensor.homepod_1_temperatur  (°C, Statistiken)       │
│     ├── sensor.homepod_1_luftfeuchtigkeit  (%, Statistiken)  │
│     └── sensor.homepod_1_absolute_luftfeuchtigkeit  (g/m³)  │
└──────────────────────────────────────────────────────────────┘
```

## Warum dieser Umweg?

- HomePods haben keine API, die HA direkt abfragen könnte
- Die Apple TV Integration macht HomeKit-Geräte in HA sichtbar, aber **nicht** die Sensordaten von HomePods (nur Mediaplayer-Funktionen)
- Apple Home kann HomePod-Sensoren auslesen — dieser Workflow nutzt Apple Home als Brücke

## Was wird einmalig angelegt?

| Objekt | Beschreibung |
|--------|--------------|
| `input_boolean.homepod_trigger` | Signal-Schalter für Apple Home |
| Trigger-Blueprint | Automation, die den Schalter alle N Minuten kurz umlegt |
| Webhook-Blueprint | Automation, die die Payload empfängt und verteilt |

## Was wird pro HomePod angelegt?

| Objekt | Beschreibung |
|--------|--------------|
| `input_number.homepod_X_temperatur` | Beschreibbarer Zwischenspeicher |
| `input_number.homepod_X_luftfeuchtigkeit` | Beschreibbarer Zwischenspeicher |
| `input_datetime.homepod_X_letzte_aktualisierung` | Zeitstempel (optional, für Dashboards) |
| `sensor.homepod_X_temperatur` | Template-Sensor mit device_class + Statistiken |
| `sensor.homepod_X_luftfeuchtigkeit` | Template-Sensor mit device_class + Statistiken |
| `sensor.homepod_X_absolute_luftfeuchtigkeit` | Berechnet aus T + RH (Magnus-Formel) |

## Warum zwei Schichten (input_number + Template-Sensor)?

Template-Sensoren sind schreibgeschützt — eine Automation kann keinen Wert direkt in einen Template-Sensor schreiben. Der `input_number` ist der einzige schreibbare Zwischenspeicher, in den eine Automation per `input_number.set_value` schreiben kann. Der Template-Sensor liest diesen Wert und verleiht ihm `device_class`, `unit_of_measurement` und `state_class` — damit werden Einheitenkonvertierung, korrekte UI-Darstellung und Langzeitstatistiken möglich.

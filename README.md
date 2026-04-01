# 🌞 PV Surplus Manager – Home Assistant Blueprint

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![HA Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue?logo=home-assistant)](blueprints/pv_surplus_manager.yaml)
[![GitHub stars](https://img.shields.io/github/stars/YOUR_GITHUB/pv-surplus-manager?style=social)](https://github.com/HatchetMan111/pv-surplus-manager)

Ein intelligenter **PV-Überschussmanager** für Home Assistant als Blueprint.  
Schaltet elektrische Verbraucher (Heizstab, Infrarotheizung, Heizlüfter, etc.) automatisch zu – abhängig vom verfügbaren PV-Überschuss, Batterie-Ladestand und PV-Forecast.

---

## ✨ Features

| Feature | Details |
|---|---|
| 🔌 Bis zu 6 Verbraucher | Frei konfigurierbar mit Namen, Leistung und Mindest-SoC |
| 🔄 Prioritätsrotation | Täglich automatisch oder manuell per `input_select` |
| 🔋 Batterie-Management | Batterie soll zum Ende des PV-Tages voll sein |
| ☀️ PV Forecast | Kompatibel mit Solcast, Forecast.Solar, eigenen Sensoren |
| ⚡ Kein Netzbezug | Alle Verbraucher werden nur mit Überschuss-Strom betrieben |
| 🛡️ Hystrese-Schutz | Verhindert häufiges Ein-/Ausschalten |
| 🚨 Notfall-Abschaltung | Alle Verbraucher aus bei kritisch niedrigem Batterie-SoC |

---

## 📋 Voraussetzungen

### Home Assistant
- Home Assistant **2023.6+**
- HACS (optional, für Solcast-Integration)

### Benötigte Sensoren
| Sensor | Beschreibung | Beispiel |
|---|---|---|
| PV Erzeugung (W) | Aktuelle PV-Leistung | `sensor.solaredge_current_power` |
| Netzbezug (W) | Positiv = Bezug, Negativ = Einspeisung | `sensor.grid_power` |
| Batterie SoC (%) | Ladestand in Prozent | `sensor.battery_soc` |
| Batterie Leistung (W) | Positiv = Laden | `sensor.battery_power` |
| Hausverbrauch (W) | Gesamtverbrauch | `sensor.house_consumption` |
| PV Forecast heute (kWh) | Vorhergesagte Tageserzeugung | `sensor.solcast_pv_forecast_today` |

### Benötigte Helfer (vor Installation erstellen!)

#### 1. `input_select` für Prioritätsreihenfolge

Gehe zu **Einstellungen → Geräte & Dienste → Helfer → + Erstellen → Auswahlliste**:

```yaml
# Oder in configuration.yaml:
input_select:
  pv_priority_order:
    name: "PV Verbraucher Priorität"
    options:
      - "heizstab,heizung1,heizung2,heizluefter"
      - "heizung1,heizung2,heizluefter,heizstab"
      - "heizung2,heizluefter,heizstab,heizung1"
      - "heizluefter,heizstab,heizung1,heizung2"
    icon: mdi:sort-variant
```

> **Wichtig:** Jede Option ist eine kommagetrennte Liste der Verbrauchernamen in der gewünschten Reihenfolge. Die Namen müssen exakt mit den im Blueprint konfigurierten Namen übereinstimmen.

---

## 🚀 Installation

### Methode 1: Direkt-Import (empfohlen)

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FHatchetMan111%2Fpv-surplus-manager%2Fblob%2Fmain%2Fblueprints%2Fpv_surplus_manager.yaml)

### Methode 2: Manuell

1. Kopiere `blueprints/pv_surplus_manager.yaml` nach:
   ```
   /config/blueprints/automation/pv_surplus_manager.yaml
   ```
2. Lade Home Assistant neu: **Entwicklerwerkzeuge → YAML neu laden**
3. Gehe zu **Einstellungen → Automationen → + Erstellen → Blueprint verwenden**
4. Suche nach **"PV Surplus Manager"**

---

## ⚙️ Konfiguration

### Schritt 1: Helfer erstellen

Erstelle zuerst den `input_select` (siehe oben).

### Schritt 2: Blueprint konfigurieren

Nach der Installation findest du im Blueprint-Dialog folgende Abschnitte:

#### ⚡ PV & Batterie Sensoren
Verknüpfe hier deine Wechselrichter- und Batterie-Sensoren.

#### ☀️ PV Forecast
- **Solcast:** `sensor.solcast_pv_forecast_forecast_today`
- **Forecast.Solar:** `sensor.energy_production_today`
- **Batteriekapazität:** Nutzbare kWh deiner Batterie
- **Ziel-SoC:** z. B. 95% – die Batterie soll bis Sonnenuntergang so voll sein
- **Puffer:** 90 Minuten vor Sonnenuntergang soll das Ziel erreicht sein

#### 🔄 Prioritätssteuerung
- Wähle deinen `input_select` Helfer
- Aktiviere die automatische tägliche Rotation (empfohlen)
- Wähle Uhrzeit der Rotation (z. B. 00:05 Uhr)

#### 🔌 Verbraucher 1–6
Für jeden Verbraucher:
- **Schalter:** Der HA-Schalter des Geräts (Shelly, Tasmota, etc.)
- **Name:** Muss mit den Einträgen im `input_select` übereinstimmen!
- **Leistung:** Nominale Wattleistung des Geräts
- **Mindest-SoC:** Batterie muss mindestens X% haben, bevor dieser Verbraucher zugeschaltet wird

#### ⚙️ Schwellen & Timing
| Parameter | Empfehlung | Beschreibung |
|---|---|---|
| Einschaltschwelle | 100–200W | Überschuss muss mind. Geräteleistung + X W betragen |
| Abschaltschwelle | 100W | Erlaubter Netzbezug bevor abgeschaltet wird |
| Prüfintervall | 30s | Wie oft wird geprüft |
| Notfall-SoC | 10% | Unter diesem Wert: alles aus |
| Batterie-Priorisierung | 50% | Unter diesem SoC: Batterie hat Vorrang vor Verbrauchern |

---

## 🔋 Batterie-Management Logik

```
┌─────────────────────────────────────────────────────┐
│              Batterie-Priorisierung                  │
├─────────────────────────────────────────────────────┤
│  SoC < Notfall-Schwelle (z.B. 10%)                  │
│    → ALLE Verbraucher AUS (Notfall)                  │
│                                                     │
│  SoC < Priorisierungsschwelle (z.B. 50%)            │
│    → PV-Überschuss geht ZUERST in die Batterie      │
│    → Verbraucher bleiben aus                         │
│                                                     │
│  Wenig Zeit bis Sonnenuntergang + Batterie nicht voll│
│    → Batterie bekommt Vorrang vor Verbrauchern       │
│                                                     │
│  Batterie auf Ziel-SoC + genug Überschuss           │
│    → Verbraucher werden nach Priorität zugeschaltet  │
└─────────────────────────────────────────────────────┘
```

### Wann bekommt die Batterie Vorrang?

1. **SoC < Priorisierungsschwelle** (z. B. 50%) – Batterie ist zu leer
2. **Wenig PV-Forecast übrig** – Vorhergesagte Restenergie reicht kaum für das Ziel
3. **< 3 Stunden bis Sonnenuntergang** – Keine Zeit mehr zum Laden

---

## 🔄 Prioritätsrotation

### Automatisch (empfohlen)

Die Automation dreht täglich (z. B. um 00:05 Uhr) den `input_select` um eine Option weiter:

```
Tag 1: heizstab → heizung1 → heizung2 → heizlüfter
Tag 2: heizung1 → heizung2 → heizlüfter → heizstab
Tag 3: heizung2 → heizlüfter → heizstab → heizung1
Tag 4: heizlüfter → heizstab → heizung1 → heizung2
Tag 5: (wieder von vorne)
```

### Manuell

Ändere den `input_select` jederzeit manuell über:
- Home Assistant UI
- Sprachsteuerung: "Setze PV Priorität auf Heizstab"
- Dashboard-Karte

### Dashboard-Karte (optional)

```yaml
type: entities
title: PV Überschussmanager
entities:
  - entity: input_select.pv_priority_order
    name: Aktuelle Priorität
  - entity: sensor.battery_soc
    name: Batterie SoC
  - entity: sensor.pv_power
    name: PV Leistung
  - entity: sensor.grid_power
    name: Netzbezug/-einspeisung
  - entity: switch.heizstab
    name: Heizstab
  - entity: switch.heizung1
    name: Infrarotheizung 1
```

---

## 🔌 Kompatible Wechselrichter / Systeme

| Hersteller | Integration | Sensoren |
|---|---|---|
| **SMA** | SMA Solar | `sensor.sma_*` |
| **Fronius** | Fronius | `sensor.fronius_*` |
| **Huawei** | Huawei Solar | `sensor.inverter_*` |
| **SolarEdge** | SolarEdge | `sensor.solaredge_*` |
| **Sungrow** | Sungrow | `sensor.sungrow_*` |
| **E3DC** | E3DC | `sensor.e3dc_*` |
| **GoodWe** | GoodWe | `sensor.goodwe_*` |
| **Kostal** | Kostal | `sensor.kostal_*` |
| **Generisch** | MQTT / REST | Eigene Sensoren |

---

## 🐛 Fehlerbehebung

### Verbraucher schaltet nicht ein
- Prüfe ob der Sensor für PV-Leistung korrekt ist
- Prüfe ob `grid_power` negativ bei Einspeisung (sonst Vorzeichen anpassen)
- Erhöhe die Einschaltschwelle temporär auf 0W zum Testen
- Prüfe den Mindest-SoC des Verbrauchers

### Verbraucher schaltet zu oft
- Erhöhe `Einschaltschwelle` und `Abschaltschwelle`
- Erhöhe das `Prüfintervall`

### Batterie wird nicht voll
- Prüfe ob `Ziel-SoC` korrekt gesetzt
- Erhöhe `Batterie-Priorisierungsschwelle`
- Erhöhe `Puffer vor Sonnenuntergang`

### Blueprint nicht sichtbar
```bash
# Home Assistant Neustart oder:
ha core restart
```

---

## 📝 Beispiel: Vollständige Helfer-Konfiguration

```yaml
# configuration.yaml

input_select:
  pv_priority_order:
    name: "PV Verbraucher Priorität"
    options:
      - "heizstab,heizung1,heizung2,heizluefter"
      - "heizung1,heizung2,heizluefter,heizstab"
      - "heizung2,heizluefter,heizstab,heizung1"
      - "heizluefter,heizstab,heizung1,heizung2"
    initial: "heizstab,heizung1,heizung2,heizluefter"
    icon: mdi:sort-variant
```

---

## 🤝 Beitragen

Pull Requests sind willkommen! Bitte erstelle zuerst ein Issue für größere Änderungen.

1. Fork das Repository
2. Erstelle einen Feature-Branch: `git checkout -b feature/meine-verbesserung`
3. Committe deine Änderungen: `git commit -am 'Add: Neue Funktion'`
4. Push zum Branch: `git push origin feature/meine-verbesserung`
5. Erstelle einen Pull Request

---

## 📄 Lizenz

MIT License – siehe [LICENSE](LICENSE)

---

## ⭐ Support

Wenn dieses Blueprint hilft, freue ich mich über einen ⭐ auf GitHub!

Fragen und Bugs bitte als [GitHub Issue](https://github.com/YOUR_GITHUB/pv-surplus-manager/issues) melden.

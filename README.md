# ESPHome Trovis 5573 Adapter

[![ESPHome](https://img.shields.io/badge/ESPHome-compatible-blue?logo=esphome)](https://esphome.io)
[![Platform](https://img.shields.io/badge/Platform-ESP32--C6-green)](https://www.espressif.com/en/products/socs/esp32-c6)
[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)

Ein ESPHome-basierter Modbus-Adapter für den **Samson TROVIS 5573** Heizungsregler.  
Der Adapter ermöglicht die vollständige Integration des Reglers in [Home Assistant](https://www.home-assistant.io/) über WLAN – ohne zusätzliche Hardware außer einem ESP32-C6.

---

## Inhalt

- [Funktionsübersicht](#funktionsübersicht)
- [Hardware](#hardware)
- [Verdrahtung](#verdrahtung)
- [Software-Voraussetzungen](#software-voraussetzungen)
- [Installation](#installation)
- [Dateistruktur](#dateistruktur)
- [Konfiguration](#konfiguration)
- [Entitäten in Home Assistant](#entitäten-in-home-assistant)
- [Status-LED](#status-led)
- [Webserver](#webserver)
- [Lizenz](#lizenz)

---

## Funktionsübersicht

- **Modbus RTU** Kommunikation mit dem TROVIS 5573 (9600 Baud, 8N1, Adresse 255)
- Auslesen von Temperaturen, Sollwerten und Betriebszuständen
- Schreiben von Parametern (Solltemperaturen, Kennlinie, etc.) über Home Assistant
- WLAN-Integration mit Fallback-Accesspoint
- Optionaler Webserver (v3) für direkten Zugriff ohne Home Assistant
- Visuelle Statusanzeige über NeoPixel-LED:
  - Grün = WLAN verbunden
  - Grün blinkend = Kein WLAN, Accesspoint aktiv
  - Rot-Flash = Modbus-Daten empfangen (RX)
  - Blau-Flash = Modbus-Daten gesendet (TX)
- OTA-Updates (Over-the-Air)

---

## Hardware

| Komponente | Beschreibung |
|---|---|
| **ESP32-C6** | Mikrocontroller (z.B. ESP32-C6 DevKit oder Seeed XIAO ESP32C6) |
| **NeoPixel (WS2812)** | 1x RGB-LED an GPIO14 |

> **Hinweis:** Der TROVIS 5573 verfügt über einen dedizierten UART-Anschluss, über den der ESP32-C6 direkt – ohne RS485-Transceiver – angeschlossen wird. Die Stromversorgung des Adapters erfolgt ebenfalls über den TROVIS 5573. Da alle Folge-Updates per OTA drahtlos eingespielt werden, ist eine galvanische Trennung nicht erforderlich.

---

## Verdrahtung

```
TROVIS 5573 (UART)        ESP32-C6
──────────────────        ────────
TX        ──────────────  GPIO3 (RX)
RX        ──────────────  GPIO2 (TX)
GND       ──────────────  GND
+3,3 V    ──────────────  VCC (3,3 V)
```

> Der UART-Anschluss des TROVIS 5573 arbeitet mit TTL-Pegel (3,3 V) und ist direkt mit dem ESP32-C6 kompatibel. Flow Control wird nicht benötigt.

---

## Software-Voraussetzungen

- [ESPHome](https://esphome.io/guides/installing_esphome) ≥ 2024.6.0
- Home Assistant (optional, aber empfohlen)
- Python 3.x (für ESPHome CLI)

---

## Installation

### 1. Repository klonen

```bash
git clone https://github.com/<dein-username>/esphome-trovis5573.git
cd esphome-trovis5573
```

### 2. WLAN einrichten (optional)

Eine WLAN-Konfiguration ist **nicht zwingend erforderlich**. Sind keine Zugangsdaten hinterlegt, öffnet der Adapter beim Start automatisch einen Accesspoint mit der SSID `ESPHome-Trovis`. Darüber lassen sich die WLAN-Credentials bequem über das Captive-Portal im Browser eingeben.

Wer die Zugangsdaten direkt in der Firmware hinterlegen möchte, legt eine `secrets.yaml` an (wird von `.gitignore` ausgeschlossen) und ergänzt `wifi:` in `esphome-trovis_5573.yaml`:

```yaml
# secrets.yaml
wifi_ssid: "Dein-WLAN-Name"
wifi_password: "Dein-WLAN-Passwort"
```

```yaml
# in esphome-trovis_5573.yaml → packages/wifi.yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

### 3. Flashen

Beim ersten Mal per USB:

```bash
esphome run esphome-trovis_5573.yaml
```

Folgeaktualisierungen per OTA:

```bash
esphome run esphome-trovis_5573.yaml --device <IP-Adresse>
```

### 4. Home Assistant

Nach dem ersten Start erscheint das Gerät automatisch in Home Assistant unter **Einstellungen → Geräte & Dienste → ESPHome** zur Integration.

---

## Dateistruktur

```
esphome-trovis5573/
├── esphome-trovis_5573.yaml       # Hauptkonfiguration – bindet alle Packages ein
└── packages/
    ├── board.yaml                 # ESP32-C6 Hardware, Framework, Logger, OTA
    ├── status_led.yaml            # NeoPixel WS2812 (GPIO14)
    ├── wifi.yaml                  # WLAN, Captive Portal, Status-LED-Logik
    ├── modbus.yaml                # UART, Modbus-Bus, Modbus-Controller, LED-Flash
    ├── trovis_sensors.yaml        # Alle Modbus-Entitäten (sensor, switch, number, ...)
    ├── webserver.yaml             # Integrierter Webserver (Port 80, Version 3)
    └── system.yaml                # Version, Reboot, Safe Mode, Factory Reset
```

Die modulare Package-Struktur erlaubt es, einzelne Funktionsblöcke einfach zu aktivieren, deaktivieren oder anzupassen, ohne die Hauptkonfiguration zu berühren.

---

## Konfiguration

Die wichtigsten Parameter befinden sich in `packages/modbus.yaml` und `packages/trovis_sensors.yaml`.

### Modbus-Adresse ändern

```yaml
# packages/modbus.yaml
modbus_controller:
  - id: trovis
    address: 255          # Standard-Broadcast-Adresse des TROVIS 5573
```

### Abfrageintervall anpassen

```yaml
# packages/modbus.yaml
modbus_controller:
  - id: trovis
    update_interval: 30s  # Alle 30 Sekunden
```

### Schreibschutz

Das **Write Enable**-Coil (Adresse 144) muss in Home Assistant aktiviert werden, bevor Parameter geschrieben werden können. Es dient als Sicherheitsschloss gegen unbeabsichtigte Änderungen.

---

## Entitäten in Home Assistant

### Sensoren (read-only)

| Name | Einheit | Modbus-Adresse | Beschreibung |
|---|---|---|---|
| Außentemperatur | °C | 9 | Gemessene Außentemperatur (AF1) |
| Vorlauftemperatur | °C | 12 | Gemessene Vorlauftemperatur (VF1) |
| Rücklauftemperatur | °C | 16 | Gemessene Rücklauftemperatur |
| Wassertemperatur | °C | 22 | Speichertemperatur (SF1) |
| Vorlaufsollwert | °C | 999 | Aktuell berechneter Vorlaufsollwert |
| Rücklauftemperatur-Begrenzung | °C | 1007 | Maximale Rücklauftemperatur |
| Systemcode-Nummer | – | 0 | Regler-Systemcode (Diagnose) |
| Firmware Version | – | 2 | Firmware-Version des TROVIS (Diagnose) |
| Hardware Version | – | 3 | Hardware-Version des TROVIS (Diagnose) |

### Binärsensoren (read-only)

| Name | Modbus-Adresse | Beschreibung |
|---|---|---|
| Umwälzpumpe | Coil 56 | Heizkreispumpe aktiv |
| Speicherladepumpe | Coil 59 | Warmwasserspeicher wird geladen |
| Zirkulationspumpe | Coil 60 | Zirkulationspumpe aktiv |

### Schalter (read/write)

| Name | Modbus-Adresse | Beschreibung |
|---|---|---|
| Write Enable | Coil 144 | Schreibschutz aufheben |
| FB05: Verz. AT-Anpassung fallend | Coil 133 | Verzögerte Außentemperaturanpassung (sinkend) |
| FB06: Verz. AT-Anpassung steigend | Coil 134 | Verzögerte Außentemperaturanpassung (steigend) |
| Fußboden-Trocknungsmodus | Coil 135 | Estrich-Trocknungsprogramm |
| Solarunterstützung | Coil 136 | Solarkreis-Integration aktiv |

### Sollwerte / Parameter (read/write)

| Name | Einheit | Adresse | Beschreibung |
|---|---|---|---|
| Solltemperatur Tag | °C | 1002 | Raumtemperatur-Sollwert Tagbetrieb |
| Solltemperatur Nacht | °C | 1003 | Raumtemperatur-Sollwert Nachtbetrieb |
| Minimale Vorlauftemperatur | °C | 1001 | Untere Vorlauftemperaturbegrenzung |
| Steigung Kennlinie | – | 1005 | Steilheit der Heizkurve |
| Niveau Kennlinie | K | 1006 | Parallelverschiebung der Heizkurve |
| Wassersollwert | °C | 1799 | Warmwasser-Solltemperatur |
| Verzögerung AT-Anpassung | K/h | 117 | Zeitkonstante für AT-Dämpfung |

---

## Status-LED

Die einzelne WS2812-NeoPixel-LED an **GPIO14** zeigt den Systemzustand auf einen Blick:

| Farbe | Muster | Bedeutung |
|---|---|---|
| 🟢 Grün | Dauerhaft | WLAN verbunden, System bereit |
| 🟢 Grün | Blinkend (0,5 Hz) | Kein WLAN – Accesspoint aktiv (`ESPHome-Trovis`) |
| 🔴 Rot | Kurzer Flash (100 ms) | Modbus-Daten empfangen (RX) |
| 🔵 Blau | Kurzer Flash (100 ms) | Modbus-Daten gesendet (TX) |

Nach jedem RX/TX-Flash kehrt die LED automatisch zum aktuellen Verbindungsstatus zurück.

---

## Webserver

Der integrierte ESPHome Webserver (Version 3) ist über `http://<IP-Adresse>` erreichbar und zeigt alle Entitäten direkt im Browser – ohne Home Assistant. Er eignet sich zur schnellen Diagnose und zum manuellen Eingriff vor Ort.

```yaml
# packages/webserver.yaml
web_server:
  version: 3
  port: 80
```

---

## Lizenz

Dieses Projekt steht unter der **Creative Commons Namensnennung – Nicht kommerziell 4.0 International (CC BY-NC 4.0)** Lizenz.

**Das bedeutet:**
- ✅ Herunterladen und privat nutzen
- ✅ Verändern und anpassen
- ✅ Weitergeben und teilen
- ✅ Eigene Projekte darauf aufbauen
- ❌ **Kommerzielle Nutzung ist nicht gestattet**

Bei Weitergabe oder Veröffentlichung von Änderungen muss der ursprüngliche Autor genannt werden.

Vollständiger Lizenztext: [https://creativecommons.org/licenses/by-nc/4.0/deed.de](https://creativecommons.org/licenses/by-nc/4.0/deed.de)

---

## Danksagung

Dieses Projekt entstand auf Basis der offiziellen [ESPHome Dokumentation](https://esphome.io) und der öffentlich zugänglichen Modbus-Registertabelle des Samson TROVIS 5573.

Besonderer Dank gilt [Tom-Bom-badil](https://github.com/Tom-Bom-badil) für die hervorragende Dokumentation der TROVIS 557x Modbus-Schnittstelle im [Samson TROVIS 557x Wiki](https://github.com/Tom-Bom-badil/samson_trovis_557x/wiki), ohne die dieses Projekt in dieser Form nicht möglich gewesen wäre.

Dank an die ESPHome- und Home-Assistant-Community für die großartige Open-Source-Arbeit.

---

*Fragen, Fehler oder Verbesserungsvorschläge? Gerne als [Issue](../../issues) oder [Pull Request](../../pulls).*

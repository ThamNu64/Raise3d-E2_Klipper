# Raise3D E2 – Klipper Conversion (IDEX)

Klipper-Konfiguration für den Umbau eines **Raise3D E2** (Independent Dual Extruder / IDEX)
auf **Klipper**. Die Steuerung basiert auf einem **Duet 2 WiFi** als Haupt-MCU und zwei
**Fly-SHT36 V3** Toolhead-Boards, die über ein **Mellow FLY-UTOR** (RS232) angebunden sind.

> ⚠️ **Umbauprojekt – kein Plug-and-Play.**
> Diese Konfiguration ist auf **meine** Hardware zugeschnitten. Pinbelegungen,
> Endschalter-Polaritäten, Motordrehrichtungen, Treiberströme und Temperaturgrenzen
> **müssen vor der ersten Inbetriebnahme geprüft werden**. Ein falsch konfigurierter
> Drucker kann Hardware beschädigen oder einen Brand verursachen. Nutzung auf eigene Gefahr.

---

## 📋 Inhaltsverzeichnis

- [Hardware](#-hardware)
- [Architektur](#-architektur)
- [Verkabelung (Kommunikation)](#-verkabelung-kommunikation)
- [Dateistruktur](#-dateistruktur)
- [Installation](#-installation)
- [Inbetriebnahme (Reihenfolge)](#-inbetriebnahme-reihenfolge)
- [IDEX-Betriebsarten](#-idex-betriebsarten)
- [LED-Statuskonzept](#-led-statuskonzept)
- [Bekannte Punkte / To-Do](#-bekannte-punkte--to-do)
- [Sicherheitshinweise](#-sicherheitshinweise)
- [Lizenz](#-lizenz)

---

## 🔧 Hardware

| Komponente | Typ | Anmerkung |
|------------|-----|-----------|
| Basisgerät | Raise3D E2 | IDEX, gemeinsames Heizbett |
| Haupt-MCU | Duet 2 WiFi (SAM4E8E) | Achsen X/Y/Z/U, Bett, Endschalter |
| Toolboard links (T0) | Fly-SHT36 V3 2SRK | Extruder, Heater, Fans, LEDs, LDC1612 |
| Toolboard rechts (T1) | Fly-SHT36 V3 2SRK | Extruder, Heater, Fans, LEDs |
| Kommunikations-Hub | Mellow FLY-UTOR | 2× RS232 (Toolboards) + 2× USB |
| Z-Probe | Fly-LDC1612 | **nur linker Kopf**, Induktiv/Eddy-Current |
| Extruder | HGX Lite (2×) | Direct Drive, NEMA14 Pancake, 1 A |
| Toolhead | DragonBurner | inkl. Nozzle- + Voron-Logo-LEDs |
| Host | Raspberry Pi | Klipper + Moonraker + Mainsail/Fluidd |
| Kamera | USB | via Crowsnest |

---

## 🧩 Architektur

```
Raspberry Pi (Klipper / Moonraker)
│
├── USB ── Duet 2 WiFi ........ Haupt-MCU
│            ├── Stepper X / Y / Z
│            ├── Stepper U (rechter IDEX-Schlitten / dual_carriage)
│            ├── Heizbett
│            └── Endschalter X / Y / Z / U
│
└── USB ── FLY-UTOR
             ├── RS232 ── SHT36 V3  (T0, links)
             │              ├── Extruder / Heater / Thermistor
             │              ├── Hotend-Fan + Part-Cooling-Fan
             │              ├── Fly-LDC1612  (Z-Homing / Bed Mesh)
             │              └── DragonBurner LEDs
             │
             └── RS232 ── SHT36 V3  (T1, rechts)
                            ├── Extruder / Heater / Thermistor
                            ├── Hotend-Fan + Part-Cooling-Fan
                            └── DragonBurner LEDs
```

Nur **24 V, GND und die RS232-Leitung** laufen durch die Schleppkette zu jedem Kopf –
Extruder, Heizung, Sensorik und Beleuchtung sitzen lokal auf dem jeweiligen SHT36.

---

## 🔌 Verkabelung (Kommunikation)

**Wichtig – FLY-UTOR ≠ FLY-UTOC:**
Der **UTOR** ist eine echte USB-/RS232-Erweiterung. Die Toolboards werden im
**RS232-Modus** betrieben (nicht CAN).

Pro SHT36 V3:
1. RS232-Firmware auf das Toolboard flashen (USB-C).
2. DIP-Schalter am Board auf RS232 stellen.
3. Gemeinsame Masse (GND) zwischen Toolboard und Host/Versorgung sicherstellen.
4. Board erscheint als serielle Schnittstelle (`CH341-UART`) → ID in die
   jeweilige `[mcu ...]`-Sektion eintragen.

> Bei Nutzung des UTOR erscheinen typischerweise **zwei** RS232-IDs. Beide
> ausprobieren und den Toolboards eindeutig zuordnen.

---

## 📁 Dateistruktur

```
printer_data/config/
├── printer.cfg .......... Haupt-Config + Includes
├── axis.cfg ............. Kinematik / Achsgrenzen / dual_carriage
├── stepper.cfg .......... Stepper-Definitionen X/Y/Z/U
├── extruder.cfg ......... Extruder T0/T1 (falls nicht in toolhead_*)
├── bed.cfg .............. Heizbett
├── sensors.cfg .......... Thermistoren / Temperatur-Sensoren
├── fans.cfg ............. Lüfter (Hotend / Part-Cooling)
├── toolhead_T0.cfg ...... Linker Kopf (Extruder, Heater, Fan, LDC1612, LEDs)
├── toolhead_T1.cfg ...... Rechter Kopf (Extruder, Heater, Fan, LEDs)
├── idex_macros.cfg ...... T0/T1, Park, Copy/Mirror, Homing
├── led.cfg .............. Status- + Arbeitslicht-Makros (DragonBurner)
├── macros.cfg ........... Allgemeine Makros
├── filament_macros.cfg .. Load / Unload / M600
├── variables.cfg ........ save_variables (Offsets etc.)
├── moonraker.conf ....... Moonraker
├── crowsnest.conf ....... Kamera (Crowsnest)
└── sonar.conf ........... Sonar (Netzwerk-Keepalive)
```

---

## 🚀 Installation

1. Klipper, Moonraker und Mainsail/Fluidd auf dem Raspberry Pi installieren
   (z. B. via [KIAUH](https://github.com/dw-0/kiauh)).
2. Klipper-Firmware für das **Duet 2 WiFi (SAM4E8E)** kompilieren und flashen.
3. RS232-Firmware auf beide **SHT36 V3** flashen, DIP auf RS232 setzen.
4. Repo in den Config-Ordner klonen:
   ```bash
   cd ~/printer_data
   git clone https://github.com/ThamNu64/Raise3d-E2_Klipper.git tmp
   cp -r tmp/printer_data/config/* ~/printer_data/config/
   ```
5. In `printer.cfg` sowie `[mcu ...]`-Sektionen die **eigenen `serial`-IDs** eintragen
   (per `ls /dev/serial/by-id/` bzw. `by-path`).
6. `FIRMWARE_RESTART` ausführen.

---

## ✅ Inbetriebnahme (Reihenfolge)

Immer **einzeln** und in dieser Reihenfolge testen:

1. **Kommunikation:** Duet + beide SHT36 werden erkannt (`RESTART`, keine MCU-Fehler).
2. **Endschalter:** `QUERY_ENDSTOPS` – Logik X / Y / Z / U prüfen (ggf. `!` am Pin).
3. **Motorrichtung:** kleine Moves je Achse; falsche Richtung → `dir_pin` invertieren.
4. **Treiberströme:** TMC-Ströme an Motoren (HGX Lite ≈ 0,65–0,75 A RMS) anpassen.
5. **Heizung:** PID-Kalibrierung Hotends + Bett (`PID_CALIBRATE`).
6. **Extruder:** `rotation_distance` je HGX Lite per 100-mm-Test kalibrieren.
7. **Z-Probe (LDC1612):** Kalibrieren, `G28 Z`, dann `BED_MESH_CALIBRATE` (nur T0).
8. **Toolchange:** `T0` / `T1` trocken testen, Parken prüfen.
9. **Tool-Offsets:** X/Y/Z-Versatz rechte Düse zur linken vermessen und eintragen.
10. **IDEX-Modi:** `COPY` und `MIRROR` erst **ohne Filament** trocken testen.

---

## 🖨️ IDEX-Betriebsarten

Gesteuert über Klippers `dual_carriage` und `SET_DUAL_CARRIAGE`
(gültige Modi: `PRIMARY`, `COPY`, `MIRROR`, `INACTIVE`):

| Makro | Funktion |
|-------|----------|
| `T0` / `T1` | Werkzeugwechsel links / rechts |
| `PARK_T0` / `PARK_T1` | Kopf an seine Parkposition |
| Normalmodus | T0 aktiv (`PRIMARY`), T1 `INACTIVE` |
| `COPY` | T1 kopiert die X-Bewegung von T0 (Duplikationsdruck) |
| `MIRROR` | T1 spiegelt die X-Bewegung von T0 |

> **Wichtig:** Vor `COPY`/`MIRROR` muss immer ein Kopf als `PRIMARY` aktiv sein.

---

## 💡 LED-Statuskonzept

Pro DragonBurner: **2× Arbeitslicht (weiß)** + **1× Voron-Logo (Status)**.
Arbeitslicht und Status sind entkoppelt – ein Statuswechsel lässt das Arbeitslicht
unangetastet und umgekehrt.

| Zustand | Status-LED (Logo) | Arbeitslicht |
|---------|-------------------|--------------|
| Idle | Blau | frei schaltbar |
| Bereit | Grün | frei schaltbar |
| Heizen (kopfbezogen) | Gelb | – |
| Drucken (kopfbezogen) | Türkis | Weiß |
| Pause / M600 | Magenta | Weiß |
| Fehler links | **Rot blinkend (links)** | **Rot blinkend (links)** |
| Fehler rechts | **Rot blinkend (rechts)** | **Rot blinkend (rechts)** |
| Globaler Fehler | **Rot blinkend (beide)** | **Rot blinkend (beide)** |

Fehlerblinken (1 Hz) läuft nicht-blockierend über `delayed_gcode` /
`UPDATE_DELAYED_GCODE` – kein `G4`, da Klipper G-Code seriell abarbeitet.

---

## 📝 Bekannte Punkte / To-Do

- [ ] `rotation_distance` beider HGX Lite final kalibrieren
- [ ] TMC-Ströme final festlegen (Motortyp bestätigen)
- [ ] Tool-Offsets T1 → T0 final vermessen
- [ ] `dual_carriage` Endschalter-Pin gegen reale Verdrahtung prüfen
- [ ] Alle `SET_DUAL_CARRIAGE`-Modi auf gültige Werte prüfen
      (`PRIMARY`/`COPY`/`MIRROR`/`INACTIVE`)
- [ ] IDEX-LED-Aufrufe gegen tatsächlich definierte Makros abgleichen
- [ ] Temperaturgrenzen (`min_temp`, `min_extrude_temp`) auf Produktivwerte setzen
- [ ] COPY / MIRROR real testen

---

## ⚠️ Sicherheitshinweise

- Für die erste Inbetriebnahme **`min_extrude_temp`** auf einen sinnvollen Wert
  (z. B. 170 °C) setzen; `0` nur bewusst und temporär für Motortests.
- **`min_temp`** nicht dauerhaft extrem niedrig lassen – das hebelt die
  Thermistor-Fehlererkennung aus.
- Heizungs-Pins (`heater_pin`) auf korrekte Invertierung prüfen, bevor real geheizt wird.
- Force-Move / Gantry-Leveling nur mit reduziertem Strom und Hand am Not-Aus.

---

## 📄 Lizenz

Sofern nicht anders angegeben, steht diese Konfiguration unter der
**GNU GPLv3** (passend zum Klipper-Ökosystem).
Klipper, Moonraker, Mainsail/Fluidd und Crowsnest unterliegen ihren jeweiligen Lizenzen.

---

## 🙏 Credits

- [Klipper](https://www.klipper3d.org/) – Kevin O’Connor & Contributors
- [Moonraker](https://github.com/Arksine/moonraker)
- [Mellow FLY](https://mellow.klipper.cn/) – UTOR / SHT36
- [Voron DragonBurner](https://github.com/chirpy2605/voron) Toolhead
- Raise3D E2 Community

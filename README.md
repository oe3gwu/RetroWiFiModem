# Retro WiFi Modem

Ein RS-232-WLAN-Modem mit Hayes-AT-Befehlen, Status-LEDs und vollem Satz an RS-232-Steuerleitungen. Firmware für ESP8266 und ESP32, plus KiCad-Platinenlayout für die ESP8266-Variante.

## Was in diesem Repository enthalten ist

| Pfad | Inhalt |
|------|--------|
| `firmware/esp8266/` | Arduino-Firmware für Wemos D1 mini |
| `firmware/esp32/` | Arduino-Firmware-Port für ESP32 |
| `kicad/esp8266/` | KiCad-Projekt (Schaltplan, Layout, Bibliotheken) |
| `kicad/esp8266/gerbers/` | Fertige Gerber-Dateien zum Bestellen der Platine |
| `kicad/esp8266/RetroWiFiModem-bom.csv` | Stückliste |
| `LICENSE.txt` | GNU GPL v3 |

## Funktionen

- RS-232-Schnittstelle (DE-9) mit TxD, RxD, RTS, CTS, DSR, DTR, DCD und RI
- Hayes-AT-Befehlssatz im WiFi232-Stil
- TCP-Verbindungen zu BBSen, Telnet-Servern und anderen Diensten
- Telnet-Protokoll: echt, fake (für bestimmte BBSen) oder deaktiviert
- 10 Kurzwahl-Slots mit Alias-Namen
- TCP-Server-Modus mit optionalem Passwort
- OTA-Firmware-Update über WLAN (Arduino IDE)

## Hardware (`kicad/esp8266/`)

Die Platine ist für einen [Wemos D1 mini](https://docs.wemos.cc/en/latest/d1/d1_mini.html) ausgelegt.

| Komponente | Funktion |
|------------|----------|
| Wemos D1 mini | ESP8266 mit WLAN |
| MAX3237 | RS-232-Pegelwandler (3,3 V ↔ ±12 V) |
| 74HCT245 | LED-Treiber für Statusanzeigen |
| 74HC32 | OR-Gatter — maskiert Boot-Ausgabe auf der seriellen Leitung |
| LM2931 | Separater 3,3-V-Regler für Peripherie |
| DFPlayer Mini | Auf der Platine vorgesehen (nicht von der Firmware in diesem Repo angesteuert) |

**Stromversorgung:** 5 V, Mittelkontakt positiv, Hohlstecker 2,1 × 5,5 mm.

**Platine bestellen:** Gerber-Dateien in `kicad/esp8266/gerbers/`, Stückliste in `kicad/esp8266/RetroWiFiModem-bom.csv`.

**Schaltplan bearbeiten:** `kicad/esp8266/RetroWiFiModem.kicad_pro` in KiCad öffnen.

### Pinbelegung (Wemos D1 mini)

Definiert in `firmware/esp8266/RetroWiFiModem.h` (ESP32-Port: `firmware/esp32/RetroWiFiModem.h`):

| Signal | GPIO | D1-mini-Pin |
|--------|------|-------------|
| CTS (Ausgang) | 15 | D8 |
| RTS (Eingang) | 13 | D7 |
| RI | 12 | D6 |
| DSR | 4 | D2 |
| DCD | 5 | D1 |
| TXEN | 14 | D5 |

> RTS/CTS sind aus Modem-Sicht (DCE) benannt.

## Firmware

Beide Varianten teilen dieselbe Modulstruktur:

```
RetroWiFiModem.ino    — Hauptschleife, Setup
RetroWiFiModem.h      — Konstanten, Pin-Definitionen
globals.h             — Globale Variablen, Einstellungsstruktur
support.h             — Hilfsfunktionen, Telnet, Verbindungslogik
at_basic.h            — Standard-AT-Befehle
at_extended.h         — Erweiterte AT-Befehle (&F, &K, &W, …)
at_proprietary.h      — Proprietäre AT-Befehle (AT$…)
```

### ESP8266 — `firmware/esp8266/`

**Arduino IDE — Voraussetzungen:**

1. Board: *LOLIN(WEMOS) D1 R2 & mini*
2. ESP8266 Core **2.7.4** (`https://arduino.esp8266.com/stable/package_esp8266com_index.json`)
3. Bibliothek [ESP_EEPROM](https://github.com/jwrw/ESP_EEPROM) **2.1.2** (ab 2.2.x schlägt `AT&W` fehl)

`firmware/esp8266/RetroWiFiModem.ino` öffnen, Board und Port wählen, kompilieren und flashen.

### ESP32 — `firmware/esp32/`

**Arduino IDE — Voraussetzungen:**

1. Board-Paket [esp32 by Espressif](https://docs.espressif.com/projects/arduino-esp32/) installieren
2. Passendes Board wählen (z. B. *ESP32 Dev Module*)

`firmware/esp32/RetroWiFiModem.ino` öffnen. GPIO-Belegung in `RetroWiFiModem.h` an eigenes Board anpassen, falls nötig.

> EEPROM-Magic-Number: ESP8266 `0x4321`, ESP32 `0x4322` — Einstellungen sind nicht zwischen Plattformen austauschbar.

## Ersteinrichtung

Werkseinstellung: **1200 Baud, 8N1**. Für die Ersteinrichtung empfiehlt sich `AT$SB=9600` und anschließend `AT&W`.

```
AT$SSID=MeinWLAN
AT$PASS=MeinPasswort
ATC1
AT&W
```

Verbindung aufbauen:

```
ATDTparticles                 ; Kurzwahl per Alias (Werkseinstellung in &F)
ATDTaltair.virtualaltair.com  ; Hostname
ATDT192.168.1.10:6400         ; IP mit Port
```

| Befehl | Beschreibung |
|--------|--------------|
| `AT$SB=n` | Baudrate (110 … 115200) |
| `AT$SU=dps` | Datenbits, Parität, Stoppbits (z. B. `8N1`) |
| `ATNETn` | Telnet: 0=aus, 1=echt, 2=fake |
| `AT&K1` | Hardware-Flowcontrol (RTS/CTS) |
| `AT$SP=n` | TCP-Server-Port für eingehende Verbindungen |
| `AT$MDNS=name` | mDNS-Name (Standard: `espmodem`) |
| `AT&Z0=host:port,alias` | Kurzwahl speichern |

Vollständige Hilfe auf dem Gerät: `AT?`

## AT-Befehle (Kurzübersicht)

Mehrere Befehle pro Zeile möglich (`AT S0=1 Q0 V1`). String-Argumente (`AT$SSID=` usw.) müssen am Zeilenende stehen.

**Verbindung:** `ATDT[+=-]host[:port]`, `ATDSn`, `ATA`, `ATH`, `ATO`, `+++` (Escape)

**WLAN:** `ATC0`/`ATC1`, `ATI`, `ATGEThttp://…`, `ATRD`/`ATRT`

**Konfiguration:** `AT&W`, `AT&F`, `AT&V0`/`AT&V1`, `AT&Zn=…`, `AT$SSID=`, `AT$PASS=`, `AT$AE=`, `AT$BM=`, `AT&R=`, `ATZ`

**Verhalten:** `ATE0`/`ATE1`, `ATQ0`/`ATQ1`, `ATV0`/`ATV1`, `ATX0`/`ATX1`, `ATS0=n`, `ATS2=n`

## OTA-Updates

Bei aktiver WLAN-Verbindung: Arduino IDE → Sketch → Upload Using Network Address.

## Bekannte Einschränkungen

- **Baudrate:** Keine Auto-Erkennung — `AT$SB` muss zum Terminal passen.
- **Linux-Telnet / Binärdateien:** Viele `0xFF`-Bytes über `telnetd` können die Verbindung abbrechen (Daemon-Problem, nicht Modem). Xmodem/Ymodem mit 128-Byte-Blöcken als Workaround.
- **ESP8266 / RTS/CTS:** Bei `AT&K1` und langem RTS-Stillstand kann der Watchdog auslösen. Die Firmware patched die UART-Sendeschleife mit `yield()`.

## Lizenz

[GNU GPL v3](LICENSE.txt). Basiert auf Virtual-Modem-Code von Jussi Salin (2016), Erweiterungen von Daniel Jameson, Stardot Contributors und Paul Rickards (2018).

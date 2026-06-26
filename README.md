# Retro WiFi Modem

Ein RS-232-WLAN-Modem mit Hayes-AT-Befehlen, Status-LEDs und vollem Satz an RS-232-Steuerleitungen.

Dieses Repository bietet zwei Wege:

| Variante | Umfang |
|----------|--------|
| **ESP8266** | Turnkey-Lösung — Firmware, KiCad-Projekt, Gerber und Stückliste |
| **ESP32-WROOM-DA** | Nur Firmware — kein Platinenlayout, eigene Hardware erforderlich |

## Was in diesem Repository enthalten ist

### ESP8266 (Turnkey)

| Pfad | Inhalt |
|------|--------|
| `firmware/esp8266/RetroWiFiModem/` | Arduino-Sketch für Wemos D1 mini |
| `kicad/esp8266/` | KiCad-Projekt (Schaltplan, Layout, Bibliotheken) |
| `kicad/esp8266/gerbers/` | Fertige Gerber-Dateien zum Bestellen der Platine |

Platine bestellen, Bauteile löten, Wemos D1 mini einstecken, Firmware flashen — fertig.

### ESP32-WROOM-DA (nur Firmware)

| Pfad | Inhalt |
|------|--------|
| `firmware/esp32/RetroWiFiModem/` | Arduino-Sketch-Port für ESP32-WROOM-DA |

Kein Schaltplan, kein Layout, keine Gerber. Die GPIO-Belegung in `firmware/esp32/RetroWiFiModem/RetroWiFiModem.h` orientiert sich an der ESP8266-Platine und muss an die eigene Verdrahtung angepasst werden.

### Allgemein

| Pfad | Inhalt |
|------|--------|
| `LICENSE.txt` | GNU GPL v3 |

## Funktionen

- RS-232-Schnittstelle (DE-9) mit TxD, RxD, RTS, CTS, DSR, DTR, DCD und RI
- Hayes-AT-Befehlssatz im WiFi232-Stil
- TCP-Verbindungen zu BBSen, Telnet-Servern und anderen Diensten
- Telnet-Protokoll: echt, fake (für bestimmte BBSen) oder deaktiviert
- 10 Kurzwahl-Slots mit Alias-Namen
- TCP-Server-Modus mit optionalem Passwort
- OTA-Firmware-Update über WLAN (Arduino IDE)

## ESP8266 — Hardware und Aufbau

Die Platine in `kicad/esp8266/` ist für einen [Wemos D1 mini](https://docs.wemos.cc/en/latest/d1/d1_mini.html) ausgelegt.

| Komponente | Funktion |
|------------|----------|
| Wemos D1 mini | ESP8266 mit WLAN |
| MAX3237 | RS-232-Pegelwandler (3,3 V ↔ ±12 V) |
| 74HCT245 | LED-Treiber für Statusanzeigen |
| 74HC32 | OR-Gatter — maskiert Boot-Ausgabe auf der seriellen Leitung |
| LM2931 | Separater 3,3-V-Regler für Peripherie |
| DFPlayer Mini | Auf der Platine vorgesehen (nicht von der Firmware angesteuert) |

**Stromversorgung:** 5 V, Mittelkontakt positiv, Hohlstecker 2,1 × 5,5 mm.

**Platine bestellen:** Gerber in `kicad/esp8266/gerbers/`.

**Schaltplan bearbeiten:** `kicad/esp8266/RetroWiFiModem.kicad_pro` in KiCad öffnen.

### Pinbelegung ESP8266 (Wemos D1 mini auf der Platine)

Definiert in `firmware/esp8266/RetroWiFiModem/RetroWiFiModem.h`:

| Signal | GPIO | D1-mini-Pin | Anbindung |
|--------|------|-------------|-----------|
| Serial TX | 1 | Tx | MAX3237 (über OR-Gatter) |
| Serial RX | 3 | Rx | MAX3237 |
| DSR | 4 | D2 | MAX3237 |
| DCD | 5 | D1 | MAX3237 |
| DTR (Eingang) | 0 | D3 | MAX3237 |
| TXEN | 14 | D5 | OR-Gatter (Boot-Müll maskieren) |
| RI | 12 | D6 | MAX3237 + LED |
| RTS (Eingang) | 13 | D7 | MAX3237 |
| CTS (Ausgang) | 15 | D8 | MAX3237 |

> RTS/CTS sind aus Modem-Sicht (DCE) benannt.

## Firmware

Beide Varianten teilen dieselbe Modulstruktur:

```
RetroWiFiModem.ino    — Hauptschleife, Setup
RetroWiFiModem.h      — Konstanten, Pin-Definitionen
globals.h             — Globale Variablen, Einstellungsstruktur
support.h             — Hilfsfunktionen, Telnet, Verbindungslogik
at_basic.h            — Standard-AT-Befehle
at_extended.h         — Erweiterte AT-Befehle (&D, &F, &K, &W, …)
at_proprietary.h      — Proprietäre AT-Befehle (AT$…)
```

### ESP8266 — `firmware/esp8266/RetroWiFiModem/`

Für die Turnkey-Platine mit Wemos D1 mini. In der Arduino IDE den Sketch-Ordner `RetroWiFiModem.ino` öffnen.

**Arduino IDE — Voraussetzungen:**

1. Board: *LOLIN(WEMOS) D1 R2 & mini*
2. ESP8266 Core **3.1.2** oder neuer (`https://arduino.esp8266.com/stable/package_esp8266com_index.json`)
3. Bibliothek [ESP_EEPROM](https://github.com/jwrw/ESP_EEPROM) in der aktuellen Version (ab 2.2.0)
4. `eeprom_storage.h` — lazy EEPROM-Init in `setup()` (kein separates Installieren nötig)

Die Firmware initialisiert EEPROM erst in `setup()` mit dem korrekten Flash-Sektor (`EEPROM_start`). Damit funktioniert `AT&W` auch mit ESP_EEPROM 2.2.x und aktuellem ESP8266-Core — die ältere Pinning-Version 2.1.2 ist nicht mehr nötig.

**Arduino IDE:** Tools → Flash Size muss zur Hardware passen (z. B. *4MB (FS:2MB OTA:~1019KB)*).

Board und Port wählen, kompilieren und flashen.

### ESP32-WROOM-DA — `firmware/esp32/RetroWiFiModem/`

Nur Software — **kein Board in diesem Repository**. Eigene Hardware mit RS-232-Pegelwandler (z. B. MAX3237) und passender GPIO-Verdrahtung erforderlich.

Die Standard-Pinbelegung in `firmware/esp32/RetroWiFiModem/RetroWiFiModem.h` entspricht der ESP8266-Platine (siehe Tabelle oben, inkl. DTR an GPIO0). Bei abweichender Verdrahtung die `#define`-Zeilen für CTS, RTS, RI, DSR, DCD, DTR und TXEN anpassen.

**Arduino IDE — Voraussetzungen:**

1. Board-Paket [esp32 by Espressif](https://docs.espressif.com/projects/arduino-esp32/) installieren
2. Board: *ESP32-WROOM-DA Module*

In der Arduino IDE den Sketch-Ordner öffnen, kompilieren und flashen.

> Die ESP8266-Platine ist **nicht** mit einem ESP32-WROOM-DA bestückbar (anderes Modul, andere Boot-Strapping-Anforderungen an GPIO 12 und 15).

> EEPROM-Magic-Number: ESP8266 `0x4321`, ESP32-WROOM-DA `0x4322` — Einstellungen sind nicht zwischen Plattformen austauschbar.

## Ersteinrichtung

Gilt für beide Firmware-Varianten. Werkseinstellung: **1200 Baud, 8N1**. Für die Ersteinrichtung empfiehlt sich `AT$SB=9600` und anschließend `AT&W`.

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
| `AT&Dn` | DTR-Verhalten: 0=ignorieren, 1=Offline, 2=auflegen, 3=Reset |
| `AT$SP=n` | TCP-Server-Port für eingehende Verbindungen |
| `AT$MDNS=name` | mDNS-Name (Standard: `espmodem` / `esp32modem`) |
| `AT&Z0=host:port,alias` | Kurzwahl speichern |

Vollständige Hilfe auf dem Gerät: `AT?`

## AT-Befehle (Kurzübersicht)

Mehrere Befehle pro Zeile möglich (`AT S0=1 Q0 V1`). String-Argumente (`AT$SSID=` usw.) müssen am Zeilenende stehen.

**Verbindung:** `ATDT[+=-]host[:port]`, `ATDSn`, `ATA`, `ATH`, `ATO`, `+++` (Escape)

**WLAN:** `ATC0`/`ATC1`, `ATI`, `ATGEThttp://…`, `ATRD`/`ATRT` (UTC über NTP, `pool.ntp.org`)

**Konfiguration:** `AT&W`, `AT&F`, `AT&V0`/`AT&V1`, `AT&Zn=…`, `AT$SSID=`, `AT$PASS=`, `AT$AE=`, `AT$BM=`, `AT&R=`, `ATZ`

**Verhalten:** `ATE0`/`ATE1`, `ATQ0`/`ATQ1`, `ATV0`/`ATV1`, `ATX0`/`ATX1`, `ATS0=n`, `ATS2=n`, `AT&D0`–`AT&D3`

## OTA-Updates

Bei aktiver WLAN-Verbindung: Arduino IDE → Sketch → Upload Using Network Address.

> Nach einem Firmware-Update mit neuen EEPROM-Feldern (z. B. `dtrHandling`) einmal `AT&W` oder `AT&F` ausführen, damit die NVRAM-Struktur passt.

## Bekannte Einschränkungen

- **Baudrate:** Keine Auto-Erkennung — `AT$SB` muss zum Terminal passen.
- **Linux-Telnet / Binärdateien:** Viele `0xFF`-Bytes über `telnetd` können die Verbindung abbrechen (Daemon-Problem, nicht Modem). Xmodem/Ymodem mit 128-Byte-Blöcken als Workaround.
- **ESP8266 / RTS/CTS:** Bei `AT&K1` und langem RTS-Stillstand kann der Watchdog auslösen. Die Firmware patched die UART-Sendeschleife mit `yield()`.

## Lizenz

[GNU GPL v3](LICENSE.txt). Basiert auf Virtual-Modem-Code von Jussi Salin (2016), Erweiterungen von Daniel Jameson, Stardot Contributors und Paul Rickards (2018).

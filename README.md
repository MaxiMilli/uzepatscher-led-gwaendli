# LED Gwändli der Uzepatscher

Dieses Dokument beschreibt die Techniken, Vorgehensweisen, Komponenten und Schnittstellen der LED-Gwändli der Uzepatscher.

**Ziel**: Auf die Musik passende und untereinander synchronisierte LED-Show.


## Inhalt

[TOC]



# Hardware

Das System umfasst hardwaretechnisch folgende Komponenten:

- 1x ein **"Tablet"** für die Steuerung
- 1x ein **"Hub"** (Raspberry Pi) für Orchestrierung
- 50x ein ESP32 als **"Nanos"** für Befehle in Lichtsignale ummünzen

## Tablet

Das Tablet kann irgend ein Gerät mit einem grösseren Bildschirm sein, dass sich via WLAN mit dem Hub verbindet und dann eine Webseite aufruft. Auf dieser Webseite können dann Befehle ausgespielt werden, welche über eine REST-Schnittstelle beim Hub Aktionen ausgelöst werden können.

- z.B. iPad 4th Generation

## Hub

Der Hub ist ein Raspberry Pi. Auf dem Hub laufen zwei Komponenten:

- Die Python-Software als Service, welche die Signale an die Nanos sendet und orchestriert
- Ein Webserver, welcher die Files für die Talbet-Applikation served.

Wichtig ist, dass der Hub eine gute WLAN-Antenne und eine gute Stromversorgung hat.

- [Raspberry Pi](https://www.digitec.ch/de/s1/product/raspberry-pi-raspberry-pi-5-4gb-entwicklungsboard-kit-38955609) mit WLAN-Antenne (unspezifiziert)
- Leistungsstarke Power-Bank

## Nano

Jeder Gugger trägt und besitzt ein Nano, welches die Singnale vom Hub via WLAN empfängt und dann in entsprechende Signale für den LED-Streifen umwandelt. Zudem natürlich einen LED-Streifen, der beliebig unterbrochen und mit 3 adrigen Lizen verlängert werdne kann. Mit Magnetverbinder ist es möglich, periphere Geräte / Accessoires zu verbinden und diese im Falle eines Unterbruches nicht zu beschädigen.

- 1x ESP32 als günstiger und robuster Mikrocontroller
  - Zur Diskussion könnte auch [dieser Mikrocontroller](https://www.reichelt.de/ch/de/arduino-nano-esp32-ohne-header-esp32-s3-usb-c-ard-nano-esp32-p353107.html?&trstct=pos_2&nbc=1) Stehen
- 1x 1.6m [LED-Streifen](https://www.bastelgarage.ch/5m-ws2812b-30led-m-led-neopixel-strip-rolle?search=WS2812B) (1/3 von 5m Rolle). Bei Bedarf können auch zwei Streifen für "innen" und "aussen" Gwändli in Anspruch genommen werden.
- 1x [Magnet-Verbindung](https://de.aliexpress.com/item/1005006531774076.html) für Hüte, Instrumente, ect.
- 2x Stecker zwischen Nano und LED-Streifen (geliefert von neuem Gugge-Präsident)
- Lizen (am Besten 3 adrig)

## Fragensammlung Hardware

- Können wir Ausrechnen, was 1.6m LED-Streifen + ESP32 mit WIFI pro 20min für Energie verbrauchen?
  - Was müssen wir der Gugge kommunizieren bezüglich Powerbanks?
- Wie sehen die Steckverbindungen bestenfalls aus?





# Software

Hauptkern des Projektes ist die Software, welche möglichst robust und sauber laufen soll.

## Landschaft

Die Software-Landschaft sieht folgendermassen aus:

- **Spezialversion MIDI-Connections**, um auf den Songs in MIDI-Struktur die Lichtshow zu planen und zu testen
- **Tablet: ``vue.js`` Applikation**, um die Songs zu orchestrieren und Kommandos zu geben
- **Hub: ``python`` Applikation/Server**, welche die Signale vom Tablet empfängt und die Sequenz-IDs via TCP/UDP an die Nanos ausspielt. Ebenfalls die Sequenzen zum Download anbietet und einen Nano willkommen heissen kann
  - ``Redis``-Server für das Echtzeit-Messageing-System, speichern der Sequenzen und überwachen/ansprechen der Nanos
  - ``daemon`` welcher auf den USB-Input hört und die darauf befindende JSON-Datei in die Sequenz-Datenbank einträgt
- **Nano: ``C++`` Applikation**, welche die Sequenz-IDs empfängt und die entsprechende Sequenz mit dem LED-Streifen abspielt

## Show-Elemente

In der Show gibt es folgende Elemente:

**Song**
Ein Song ist ein abgeschlossenes Lied, so wie wir es in der Gugge kennen

**Part**
Ein Song hat mehrere Parts wie Refrain, Strophe, Bridge, ect.

**Sequenz**
Einen Part besteht aus einer oder mehreren Sequenzen, die in einem bestimmten Interval nacheinander abgespielt werden. Eine Sequenz kann ein Fade in-out oder blinken oder einfach steady Licht sein

## Informationshierarchie

Was wissen und orchestrieren die einzelnen Komponenten?

**Tablet**
[Besitzt keine nennenswerten Informationen]

**Hub**
- Zuordnung Nano-ID zu
  - IP-Adresse
  - Gugger
  - Register
- Alle Songs und deren Parts und deren Sequenzen

**Nano**
- Hat eine fixe ID
- Hat alle möglichen Sequenzen temporär gespeichtert (bis nächsten Neustart)

## Requirements & Schnittstellen

Dieser Abschnitt beschreibt, was die Elemente im Detail können müssen inkl. der Schnittstellen

### Hub

- Der Hub hat eine fixe 'masterIP': `192.168.220.1`

#### Hotspot

Der Hub etabliert einen Hotspot, mit welchem sich die Nanos und das Tablet verbinden können. Daten dafür:
- SSID: `uzepatscher_lichtshow`
- Passwort: `NZpqLB2kqg`


#### Datenbank

Die Redis-Datenbank speichert folgende Informationen strukturiert:

**musicians**
| mac_address     | ip             | uzepatscher | register |
| --------------- | -------------- | ----------- | -------- |
| 25:df:we:ep:32: | 192.168.220.24 | Samuel R.   | Drums    |

**songs**
song |  
----------- |
Killing in the Name | 

#### Endpunkte

Der Hub stellt folgende Endpunkte zur Verfügung

**für Nanos**
`GET /nano/sequences` liefert in JSON-Format alle verfügbaren Sequenzen zrück.
`GET /nano/register` Nano registriert sich.
`GET /nano/heartbeat` Nano zeigt, dass er am Leben ist.

**für Tablet**
`GET /tablet/sequences` liefert in JSON-Format alle verfügbaren Sequenzen zrück.
`GET /tablet/songs` liefert in JSON-Format alle verfügbaren Songs
`GET /tablet/queue` liefert in JSON-Format die aktuelle Queue zurück
`POST /tablet/queue` Song zur Queue hinzufügen
`DELETE /tablet/queue` Song von Queue entfernen
`POST /tablet/tap` In einen Song-Part tapen (z.B. von Refrain in Strophe)
`POST /tablet/command` Manueller Command an alle Nanos senden




### Nano

- Der Nano hat eine Mac-Adresse, mit welcher er sich identifiziert (anonym und nicht auf einen Uzepatscher bezogen -> austauschbarkeit)

**WIFI**

Sobald der Nano am Strom ist, sucht er in kurzen Abständen (ca. 2-3s) nach dem WIFI des Hubs in der Umgebung.

Die Suchabstände soll aus Energiespargrünen konstant verlängert werden. Nach
- 20 Versuchen, Abstand auf 20s
- 40 Versuchen, Abstand auf 1m

Bei jedem Loop überprüft der Nano, ob er noch mit dem WiFi verbunden ist und falls nicht, startet er wieder den Suchvorgang.

**Anmeldung**

Sobald das WIFI gefunden wurde oder die Verbindung wieder etabliert ist, passiert eine Anmeldung beim Hub. Damit wird die IP-Adresse (neu) gespeichert und der Nano kann angesprochen werden.

`TPC: masterIP` Mit einem TPC-Kommando wird als Payload die Mac-Adresse übergeben

**Initialisieren**

Sobald angemeldet, initialisiert sich der Nano. Er fetcht alle möglichen Sequenzen vom Hub von diesem Endpunkt: `/nano/sequences`

**Standby**
Wenn der Nano nach 3 Versuchen keine WIFI-Verbindung aufbauen konnte, soll er in den Standby-Mode wechseln. Das können folgende Modis sein:

- Lauflicht im Pingpong in einem der drei UP-definierten Farbbereichen
- Random fade-in-out in zufälligen 

Die Leuchtstärke ist bei maximal 10% um möglichst Energie zu sparen.

Wenn sich der Nano mit dem WLAN verbindet, soll er den Standby-Mode weiterführen, bis er ein Kommando erhält

Auf das Standby-Kommando soll der Nano in den Standby fallen und die Sequenz nicht unterbrechen, wenn sich das WIFI trennt.

**Sequenzen**

(Hauptteil)
Der Nano parsed in jedem Loop die TCP-Pakete und interpretiert das Kommando wie z.b. `ACT-xxxxxxxxxx`

Das Kommando hat immer eine Dauer (entweder definiert in der Sequenz oder im Kommando selber). Nach dieser Dauer soll der Nano auf nächste Anweisungen warten.

**Heartbeat**

(Dies ist bitte noch zu challengen, ob das sinn macht)
Wenn der Nano mit dem WIFI verbunden ist, gibt er jede Minute einen Heartbeat an den Hub mit einem `TPC`-Paket. Grund: Sobald der Hub einen Nano seit 2-3 min nicht mehr gesehen hat, ist dieser offline (-> wichtig für )

### MIDI Format für LED RGB Streifen Steuerung

Edi wird in MIDI Connections ein Interface einbauen, welches es einfach macht, die Lichtshow zu kreieren, zu konstruieren und zu organisieren. Es gibt mehrere Kanäle mit je 128 "Noten". Die Noten haben jeweils zwei Werte: Länge (Duration) und "Velocity" (dieser Wert kann aber unterschiedlich sein). Die verschiedenen Kanäle werden nur gebraucht, wenn sich mehrere Effekte überlagern.

**Register**

Wenn sich das Register matcht, dann werden alle anderen Werte "applied".

| Kanal | Beschriftung      | Beschreibung       | Duration | Velocity | Bemerkungen |
| ----- | ----------------- | ------------------ | -------- | -------- | ----------- |
| 1     | Ganze Gugge       | überschreibt alles | Aktiv    | -        |
| 2     | Drums             |                    | Aktiv    | -        |
| 3     | Wäggeli           |                    | Aktiv    | -        |
| 4     | Pauken            |                    | Aktiv    | -        |
| 5     | Lira              |                    | Aktiv    | -        |
| 6     | Chinellen         |                    | Aktiv    | -        |
| 7     | 1. Trompete       |                    | Aktiv    | -        |
| 8     | 2. Trompete       |                    | Aktiv    | -        |
| 9     | 1. Posaune        |                    | Aktiv    | -        |
| 10    | 2. Posaune        |                    | Aktiv    | -        |
| 11    | 3. Posaune        |                    | Aktiv    | -        |
| 12    | Bässe             |                    | Aktiv    | -        |
| 13    | Bässe Instrumente |                    | Aktiv    | -        |

**Farben**

| Kanal | Beschriftung | Beschreibung | Duration | Velocity             | Bemerkungen          |
| ----- | ------------ | ------------ | -------- | -------------------- | -------------------- |
| 16    | R            | Rot          | Dauer    | Lichtstärke 0 - 100% |
| 17    | G            | Grün         | Dauer    | Lichtstärke 0 - 100% |
| 18    | B            | Blau         | Dauer    | Lichtstärke 0 - 100% |
| 19    | RB           | Regenbogen   | Dauer    | Lichtstärke 0 - 100% | Geschwindigkeit = 3s |

**Einstellungen**

Diese Einstellungen können andere Effekte beeinflussen.

| Kanal | Beschriftung | Beschreibung                      | Duration           | Velocity             | Bemerkungen                            |
| ----- | ------------ | --------------------------------- | ------------------ | -------------------- | -------------------------------------- |
| 25    | Speed        | Geschwindigkeit des Effekts in ms | Gültigkeitsbereich | Speed 0 - 10'000ms   |
| 26    | Länge        | Länge des Effekts in Pixel        | Gültigkeitsbereich | Anzahl LED 0 - 100   |
| 27    | Lichtstärke  | Intensität des Lichts             | Gültigkeitsbereich | Lichtstärke 0 - 100% | überschreibt alle anderen Lichtstärken |

**Effekte**

Wenn kein Effekt angewählt ist, strahlen die LEDs einfach die gewählte Farbe. Effekte sind nicht mischbar, es wird der erste Kanal genommen, welcher verfügbar ist

| Kanal | Beschriftung | Beschreibung                     | Duration | Velocity             | Bemerkungen                    |
| ----- | ------------ | -------------------------------- | -------- | -------------------- | ------------------------------ |
| 30    | Lauflicht    | Tatzelwurm an LED-Leuchten       | Dauer    | Lichtstärke 0 - 100% | Speed, Länge                   |
| 31    | Glitzern     | Zufälliges Blinken aller LED     | Dauer    | Lichtstärke 0 - 100% |
| 32    | Welle        | Wellenbewegung entlang der LEDs  | Dauer    | Lichtstärke 0 - 100% | Speed, Länge                   |
| 33    | Pulsieren    | LED pulsiert an/aus              | Dauer    | Lichtstärke 0 - 100% | Speed                          |
| 34    | Farbwechsel  | Wechsel der Farben über Zeit     | Dauer    | Lichtstärke 0 - 100% | Speed                          |
| 35    | Strobo       | Stroboskop-Effekt                | Dauer    | Lichtstärke 0 - 100% | Speed                          |
| 36    | Fade         | Langsames Ein- und Ausblenden    | Dauer    | Lichtstärke 0 - 100% | Speed                          |
| 37    | Herzschlag   | Simulation eines Herzschlags     | Dauer    | Lichtstärke 0 - 100% | Speed                          |
| 38    | Meteor       | Lichtkugeln ziehen über die LEDs | Dauer    | Lichtstärke 0 - 100% | Geschwindigkeit, Länge (Pixel) |
| 39    | Flackern     | LEDs flackern wie Kerzenlicht    | Dauer    | Lichtstärke 0 - 100% |

**Guggen-Übergreifende Effekte**

| Kanal | Beschriftung          | Beschreibung                                                | Duration | Velocity             | Bemerkungen       |
| ----- | --------------------- | ----------------------------------------------------------- | -------- | -------------------- | ----------------- |
| 50    | Random                | Zufällige(r) Gugger Leuchten mit gewähltem Effekt           | Dauer    | Lichtstärke 0 - 100% | Länge (wie viele) |
| 51    | Wellenüberlauf        | Wellenbewegung von links nach rechts in optimaler Formation | Dauer    | Lichtstärke 0 - 100% | Speed             |
| 52    | Asynchrones Pulsieren | Alle Gugger pulsieren asynchron                             | Dauer    | Lichtstärke 0 - 100% | Speed             |
| 53    | Farbexplosion         | Zufälliger Farbwechsel bei allen Guggern gleichzeitig       | Dauer    | Lichtstärke 0 - 100% | Speed             |




### Kommandos

Kommandos sind die Pakete (Strings), die per **UDP** von Hub an Nanos übertragen wird.

Die Kommandos sind so aufgebaut: `XXX-DDDDDDDD-00`
- `XXX` drei Buchstaben geben an, welcher Modus angesprochen werden soll
- `DDDDDDDDDDDD` kann 
  - einen Farbcode (3x Zahl zwischen 0 und 256) im Format "RGB" + Leuchtstärke sein (12-stellige Zahl)
  - eine Sequenz-ID sein (8-stellig)
  - ein aktionsspezifisches Format haben
- `00` ist die Dauer für das Anzeigen des Kommandos. 00 = unlimitiert

Es gibt folgende fixen Kommandos

- `OFF` schaltet den Nano aus
- `SBY` schaltet den Nano in den Standby
- `RGB-123123123256-00` schaltet den Nano für x Zeit in eine x beliebige Farbe mit x beliebiger Leuchtstärke
- `SEQ-asdfasdf-20` spielt die Sequenz 'asdfasdf' ab. Wenn kein weiteres Kommando kommt, wird die Sequenz 20s oder gemäss Sequenz-Daten abgespielt.
- `TAP-250-00` lässt das LED 0 in 250 BPM blinken
- `RBW-0000.256-00` spielt den Regenbogen ab. `0000` ist die Geschwindigkeit und `256` die Helligkeit.
- `WLK-123123123256.0000.00-00` spielt ein Lauflicht ab. `0000` ist die Geschwindigkeit und `00` die Länge in Pixel (bis nächste Welle kommt).
- `FDE-123123123256.0000-00` schaltet den Nano in einen Fade-Modus (Leuchtstärke ist max. Fade). `0000` gibt die Geschwindigkeit in Millisekunden an (von 0 - x)
- `SIN-123123123256.00.0-00` schaltet ein einzelnes Pixel in einer entsprechenden Farbe an. `00` gibt den Pixel-Index an und `0` ist ein boolean, wenn 0 = wird von nächstem SIN-Kommando überschrieben, 1 = wird fix die Zeit abwarten unabhängig vom nächsten SIN-Kommando.


# Infos an Gugge

Um uns schon mal vorzubereiten und zu sammeln, was wir alles bezüglich der technischen Seite der Gugge mitteilen müsse.

- ESP-32 soll möglichst Feuchtigkeitsresistet eingepackt werden (z.b. Plastiksäckli), aber nicht mehrschichtig wegen WLAN.
- Jeder Gugger bekommt 1.6m LED-Streifen. Dieser kann überall getrennt und mit Drähte irgendwo weiter verbunden werden. Wer will, darf für "innen" und "aussen" einen zweiten Streifen beanspruchen, es soll aber immer nur einer aufs Mal leuchten
- Es kann im Streifen eine Magnet-Verbindung eingebaut werden, welche z.B. zu einem Hut oder Instrument geht (und so einfach getrennt werden kann)
- Powerbank-Grösse (siehe Frage oben)

# AP3 – Steuerungskonzept

**Projekt:** MUSV 2026/27 – Miniaturisiertes unbemanntes Wasserfahrzeug  
**Hochschule:** Ernst-Abbe-Hochschule Jena  
**Arbeitspaket:** AP3 – Mikrocontroller und Steuerung  
**Verantwortlich:** Jiangnan Wu  
**Datum:** 22.10.2026  
**Version:** 0.1 – Vorläufiger Konzeptentwurf

---

## 1. Zielsetzung

Ziel des Arbeitspakets AP3 ist die Konzeption, Implementierung und Erprobung der Steuerung eines miniaturisierten unbemannten Wasserfahrzeugs (MUSV).

Das Wasserfahrzeug soll über eine Funkfernsteuerung manuell bedient werden können. Folgende Funktionen sind vorgesehen:

- Vorwärtsfahrt und Rückwärtsfahrt
- Änderung der Fahrtrichtung durch unterschiedliche Ansteuerung der beiden Antriebsmotoren
- Stillstand bei neutraler Steuerstellung
- Aktivierung und Deaktivierung des Antriebs
- Sicherer Stopp der Antriebe bei Verlust des Funksignals

Zusätzlich sollen GPS- und Kompassdaten erfasst werden. Eine autonome Navigation ist im aktuellen Projektumfang nicht vorgesehen.

Als zentrale Steuereinheit wird der **Matek H7A3-SLIM Flight Controller** eingesetzt.

Das vorliegende Dokument beschreibt das vorläufige Steuerungskonzept. Die konkrete Umsetzung wird nach Auswahl und Überprüfung aller erforderlichen Hardwarekomponenten festgelegt.

## 2. Systemarchitektur

Das Steuerungssystem besteht aus einer Funkfernsteuerung, einem Funkempfänger, dem Hauptcontroller sowie zwei getrennt ansteuerbaren Antrieben.

### 2.1 Signalfluss

**Fernsteuerung → Funkempfänger → SBUS → Matek H7A3-SLIM → zwei Motorregler → zwei Antriebsmotoren**

Die Funkfernsteuerung überträgt die gewünschten Fahrbefehle an den Empfänger. Dieser stellt die empfangenen Kanaldaten über die serielle SBUS-Schnittstelle bereit.

Der Mikrocontroller verarbeitet die Steuerbefehle und erzeugt entsprechende Ausgangssignale für die beiden Motorregler. Durch die unabhängige Ansteuerung der Motoren werden Geschwindigkeit und Fahrtrichtung des Wasserfahrzeugs beeinflusst.

Die Sensorik ist über separate Schnittstellen mit dem Hauptcontroller verbunden:

- GPS-Modul: serielle Kommunikation über UART
- Kompass: voraussichtlich I²C
- Weitere Sensoren: bei Bedarf über freie Schnittstellen

Die Sensorsignale dienen zunächst ausschließlich der Datenerfassung und gegebenenfalls der Protokollierung.

## 3. Hardwarekonzept

### 3.1 Hauptcontroller – Matek H7A3-SLIM

Der Matek H7A3-SLIM basiert auf dem STM32H7A3RIT6 mit einem ARM-Cortex-M7-Prozessor.

Für das Projekt sind insbesondere folgende Eigenschaften relevant:

| Eigenschaft | Technische Daten |
|---|---|
| Mikrocontroller | STM32H7A3RIT6 |
| Prozessortakt | 280 MHz |
| Flash-Speicher | 2 MB |
| UART-Schnittstellen | 6 |
| PWM-Ausgänge | 11 |
| I²C | 1 |
| CAN | 1 |
| USB | USB-C |
| Interne Sensorik | Beschleunigungssensor/Gyroskop, Barometer |
| Externer Kompass | Über I²C anschließbar |

Die vorhandenen Schnittstellen ermöglichen den Anschluss des Funkempfängers, der Motorregler und der vorgesehenen Sensorik.

Als Firmware wird **ArduPilot Rover** vorgesehen. Für den Controller ist ein entsprechender Firmware-Build verfügbar.

### 3.2 Funkfernsteuerung und Empfänger

Als Referenz für das Fernsteuerungssystem dienen die Unterlagen der FrSky Taranis Q X7/X7S und des FrSky RX6R.

Die Kombination ermöglicht die Übertragung mehrerer Steuerkanäle über eine 2,4-GHz-Funkverbindung.

Der RX6R-Empfänger unterstützt SBUS zur seriellen Übertragung von bis zu 16 Steuerkanälen. Für die geplante Anwendung werden mindestens zwei analoge Steuerfunktionen und eine zusätzliche Sicherheits- beziehungsweise Betriebsfunktion benötigt.

Die Kompatibilität der tatsächlich verwendeten Sender- und Empfänger-Firmware (ACCESS bzw. ACCST D16) muss vor der Inbetriebnahme geprüft werden.

Die endgültige Auswahl des Fernsteuerungssystems wird nach Abstimmung innerhalb der Projektgruppe bestätigt.

### 3.3 Motorregler und Antriebsmotoren

Das Wasserfahrzeug soll durch zwei getrennt ansteuerbare Motoren angetrieben werden.

Jeder Motor benötigt eine passende Leistungselektronik, beispielsweise einen bidirektionalen elektronischen Drehzahlregler (ESC).

Die Motorregler müssen folgende Funktionen ermöglichen:

- Einstellbare Motorleistung
- Vorwärts- und Rückwärtsbetrieb
- Definierter Neutral- bzw. Stillstandszustand
- Kompatibilität mit dem Ausgangssignal des Controllers

Die Auswahl der Motoren und Motorregler erfolgt in Abstimmung mit den zuständigen Teammitgliedern.

## 4. Kommunikationsschnittstellen

### 4.1 SBUS-Eingang

Zur Übertragung der Steuerbefehle wird das digitale SBUS-Protokoll vorgesehen.

Das Protokoll verwendet eine serielle Übertragung mit folgenden Eigenschaften:

| Parameter | Wert |
|---|---|
| Baudrate | 100000 Baud |
| Datenbits | 8 |
| Parität | Even |
| Stoppbits | 2 |
| Signalform | Invertiertes serielles Signal |
| Datenrahmen | 25 Byte |

Ein SBUS-Datenrahmen enthält bis zu 16 analoge Steuerkanäle sowie Statusinformationen, beispielsweise zur Erkennung eines Signalverlusts.

Die Referenzsoftware `SBUS.py` des früheren USV-Projekts enthält bereits einen Decoder für dieses Datenformat. Dabei werden unter anderem Kanal 1 für den Schub, Kanal 2 für die Lenkung und Kanal 14 für eine zusätzliche Steuerfreigabe verwendet.

Diese Kanalbelegung kann als Orientierung dienen, muss jedoch für das neue Projekt mit der Konfiguration von ArduPilot abgestimmt werden.

### 4.2 Integration in den Matek-Controller

Beim Matek H7A3-SLIM ist die serielle RC-Eingangsschnittstelle standardmäßig USART2 zugeordnet.

**Vorgesehener Anschluss:**

- SBUS-Signal des Empfängers → RX2
- Versorgung des Empfängers → geeignete Versorgungsspannung
- Masse des Empfängers → GND

Für ArduPilot sind folgende Parameter vorgesehen:

| Parameter | Wert | Funktion |
|---|---|---|
| `SERIAL2_PROTOCOL` | `23` | RC-Eingang |
| `SERIAL2_OPTIONS` | `3` | SBUS-Signalinvertierung |

Da ArduPilot die Dekodierung des SBUS-Protokolls bereits unterstützt, ist eine separate Implementierung des früheren Python-Decoders zunächst nicht notwendig.

Die frühere Implementierung verwendet die PIO-Einheit des RP2040. Dieser Programmteil ist nicht direkt auf den STM32H7A3 übertragbar.

### 4.3 Vorgesehene Schnittstellenbelegung

| Funktion | Matek-Schnittstelle | Status |
|---|---|---|
| Funkempfänger (SBUS) | RX2 / USART2 | Vorgesehen |
| Linker Motorregler | S1 / PWM1 | Vorgesehen |
| Rechter Motorregler | S3 / PWM3 | Vorgesehen |
| GPS | TX3/RX3 / USART3 | Vorgesehen |
| Kompass | SDA/SCL / I²C | Vorgesehen |
| Konfiguration und Diagnose | USB-C | Vorgesehen |

Die Angaben stellen die geplante Schnittstellenbelegung dar. Die elektrische Kompatibilität ist vor dem Anschluss der Komponenten zu überprüfen.

## 5. Steuerungsprinzip

### 5.1 Differentialsteuerung

Die Fahrtrichtung des Wasserfahrzeugs soll durch unterschiedliche Schubkräfte der beiden Antriebsmotoren verändert werden.

Dieses Prinzip wird als **Differentialsteuerung (Differential Thrust bzw. Skid Steering)** bezeichnet.

Beide Antriebe können unabhängig voneinander vorwärts oder rückwärts betrieben werden.

Das Verhalten lässt sich durch zwei normierte Steuergrößen beschreiben:

- \(u\): gewünschter Schub
- \(r\): gewünschte Lenkbewegung

Ein vereinfachtes mathematisches Modell lautet:

\[
u_L=\operatorname{clip}(u-kr,-1,1)
\]

\[
u_R=\operatorname{clip}(u+kr,-1,1)
\]

Dabei sind \(u_L\) und \(u_R\) die Sollwerte für den linken und rechten Antrieb. Der Faktor \(k\) beschreibt den Einfluss des Lenkbefehls.

Die Begrenzungsfunktion `clip` stellt sicher, dass die zulässigen Stellgrößen nicht überschritten werden.

Die tatsächliche Drehrichtung wird bei der Inbetriebnahme geprüft.

### 5.2 Fahrzustände

| Fahrbefehl | Linker Antrieb | Rechter Antrieb |
|---|---|---|
| Vorwärts | Vorwärts | Vorwärts |
| Rückwärts | Rückwärts | Rückwärts |
| Geradeaus | Gleicher Schub | Gleicher Schub |
| Kurve | Unterschiedlicher Schub | Unterschiedlicher Schub |
| Drehung auf der Stelle | Gegensinniger Schub | Gegensinniger Schub |
| Stopp | Neutral | Neutral |

Die Drehung auf der Stelle setzt voraus, dass beide Antriebe bidirektional betrieben werden können.

### 5.3 Referenzalgorithmus

Im früheren USV-Projekt wurde das Modul `drive_system.py` zur Berechnung der Stellwerte zweier Antriebe eingesetzt.

Die Klasse `DriveSystem` verwendet dazu die Variablen `trust`, `rudder` und `hysterese`.

Der Algorithmus berücksichtigt einen Bereich um die neutrale Schubstellung und erzeugt unterschiedliche Sollwerte für die beiden Antriebe.

Die grundlegende Idee wird für das neue Projekt übernommen. Die Implementierung soll jedoch zunächst über die bereits vorhandene Skid-Steering-Funktion von ArduPilot Rover erfolgen.

Eine eigene Implementierung ist nur vorgesehen, falls die Standardfunktionen die Projektanforderungen nicht ausreichend erfüllen.

### 5.4 Umsetzung in ArduPilot Rover

Für die getrennte Ansteuerung der beiden Motoren werden folgende Ausgangsfunktionen vorgesehen:

| Parameter | Wert | Bedeutung |
|---|---|---|
| `SERVO1_FUNCTION` | `73` | Throttle Left |
| `SERVO3_FUNCTION` | `74` | Throttle Right |

Die beiden zugehörigen Ausgänge werden mit den Motorreglern verbunden.

Für den ersten Test wird der Betriebsmodus **Manual** verwendet. In diesem Modus kann das Fahrzeug ohne GPS-basierte Positionsregelung manuell gesteuert werden.

Die Zuordnung der Steuerkanäle erfolgt über die Senderkonfiguration und gegebenenfalls die RCMAP-Parameter. Die Neutralstellungen und Stellbereiche müssen anschließend kalibriert werden.

### 5.5 Ansteuerkennlinie der Motoren

Die Referenzunterlage `Steuerkennlinie Haswing Protruar.pdf` beschreibt eine stückweise definierte Kennlinie zur Umsetzung von Schub-Sollwerten in Motorstellwerte.

Darin sind getrennte Bereiche für Vorwärtsfahrt, Stillstand und Rückwärtsfahrt vorgesehen.

Das Prinzip einer solchen Kennlinie kann verwendet werden, um die gewünschte Motorleistung an das tatsächliche Verhalten der Antriebe anzupassen.

Die dokumentierten Zahlenwerte beziehen sich jedoch auf die frühere Hardware und werden nicht unmittelbar übernommen.

Die endgültige Kennlinie sowie Neutralpunkt, PWM-Signalbereich und maximale Stellwerte müssen anhand der neu ausgewählten Motorregler festgelegt werden.

## 6. Sicherheitskonzept

Ein sicherer Betrieb des Wasserfahrzeugs setzt voraus, dass unerwartete Motorbewegungen möglichst verhindert werden.

### 6.1 Funksignalverlust – Failsafe

Bei Verlust der Funkverbindung darf das Fahrzeug nicht dauerhaft mit dem zuletzt empfangenen Schubbefehl weiterfahren.

Daher wird eine Failsafe-Funktion vorgesehen, die bei fehlendem oder ungültigem RC-Signal die Antriebe in den sicheren Neutralzustand überführt.

Für ArduPilot Rover sind zunächst folgende Einstellungen vorgesehen:

- `FS_THR_ENABLE = 1` – Radio-Failsafe aktivieren
- `FS_ACTION = 2` – Wechsel in den Hold-Modus
- `FS_TIMEOUT` – Zeitgrenze nach Prüfung der tatsächlichen Signalübertragung festlegen

Im Hold-Modus sollen beide Motoren ohne Schub bleiben. Dazu müssen die Ausgänge und die Neutralwerte der Motorregler entsprechend kalibriert werden.

Die tatsächliche Failsafe-Funktion wird vor dem ersten Einsatz auf dem Wasser getestet.

### 6.2 Aktivierung und Deaktivierung

Die Motoren dürfen erst nach einer bewussten Freigabe gestartet werden.

Hierzu wird die Arm-/Disarm-Funktion von ArduPilot eingesetzt. Eine zusätzliche Schalterbelegung auf der Funkfernsteuerung ist vorgesehen.

Die endgültige Kanalzuordnung wird während der RC-Konfiguration festgelegt.

### 6.3 Weitere Schutzmaßnahmen

Für die spätere Implementierung werden folgende Schutzmaßnahmen berücksichtigt:

- Prüfung der Versorgungsspannungen
- Begrenzung der Motorstellwerte
- Überprüfung der Neutralstellung vor dem Aktivieren
- Sichere Verkabelung und gemeinsame Signalmasse
- Schutz der Elektronik gegen Feuchtigkeit
- Prüfung der Motorsteuerung ohne montierte Propeller

## 7. Sensorintegration

Das Projekt sieht die Erfassung von GPS- und Kompassdaten vor.

Für den Matek H7A3-SLIM kann ein GPS-Modul beispielsweise über USART3 angeschlossen werden. Ein externer Kompass kann über die I²C-Schnittstelle eingebunden werden.

Die Messwerte sollen zunächst zur Anzeige beziehungsweise Protokollierung zur Verfügung stehen.

Eine automatische Kursregelung, Wegpunktnavigation oder Hindernisvermeidung ist nicht Gegenstand der aktuellen Projektphase.

Die Auswahl der Sensoren und die Festlegung der Datenschnittstellen erfolgen in Abstimmung mit dem zuständigen Arbeitspaket für Sensorik.

## 8. Inbetriebnahme und Testkonzept

Die Implementierung soll schrittweise erfolgen.

| Test | Durchführung | Erfolgskriterium |
|---|---|---|
| T1: Controller | Rover-Firmware installieren und verbinden | Controller wird erkannt |
| T2: RC-Empfang | Sender und Empfänger verbinden | Steuerkanäle werden erkannt |
| T3: Neutralstellung | Knüppel in Neutralstellung | Beide Motorbefehle neutral |
| T4: Vorwärts/Rückwärts | Schubbefehl ändern | Beide Antriebe reagieren korrekt |
| T5: Lenkung | Lenkbefehl ändern | Unterschiedliche Motorstellwerte |
| T6: Failsafe | Sender ausschalten | Beide Antriebe stoppen |
| T7: Sensorik | GPS/Kompass anschließen | Messwerte verfügbar |

Die Tests T1 bis T6 werden zunächst am Prüfstand ohne montierte Propeller durchgeführt.

Eine Erprobung auf dem Wasser erfolgt erst nach erfolgreicher Prüfung der sicherheitsrelevanten Funktionen.

## 9. Offene technische Fragen

Vor der endgültigen Umsetzung müssen folgende Fragen geklärt werden:

1. Welche Motoren und Motorregler werden tatsächlich eingesetzt?
2. Unterstützen die Motorregler einen bidirektionalen Betrieb?
3. Welche Spannungen und Steuerprotokolle benötigen die Motorregler?
4. Werden die FrSky Taranis Q X7/X7S und der RX6R aus dem Referenzprojekt übernommen?
5. Welche GPS- und Kompassmodule werden verwendet?
6. Welche Kanalbelegung der Fernsteuerung ist für die Bedienung am sinnvollsten?
7. Welche Parameter und Neutralwerte gewährleisten einen sicheren Stopp bei Signalverlust?

Die Ergebnisse dieser Klärungen werden in der nächsten Version des Steuerungskonzepts ergänzt.

## 10. Nächste Arbeitsschritte

Nach Abschluss der Konzeptphase sind die folgenden Arbeitsschritte geplant:

**Phase 1 – Vorbereitung**

- Auswahl und Prüfung der benötigten Hardware
- Erstellung eines endgültigen Anschlussplans
- Vorbereitung der Konfigurationssoftware

**Phase 2 – Implementierung**

- Installation der ArduPilot-Rover-Firmware
- Einrichtung des SBUS-Empfangs
- Konfiguration der beiden Motorenausgänge
- Kalibrierung der Fernsteuerung und Motorregler

**Phase 3 – Validierung**

- Funktionstests der einzelnen Antriebe
- Überprüfung der Differentialsteuerung
- Test der Failsafe- und Arm-/Disarm-Funktionen
- Integration in das Gesamtfahrzeug

## 11. Verwendete Referenzen

**Unterlagen des früheren USV-Projekts von Prof. Grabow:**

1. `X7 X7S ACCESS Manual.pdf` – FrSky Fernsteuerung
2. `RX6R ACCST-Manual-1.pdf` – FrSky Empfänger
3. `SBUS.py` – SBUS-Empfang und Dekodierung
4. `UART_RX_Dokumentation.pdf` – UART-Empfang mit RP2040-PIO
5. `Steuerkennlinie Haswing Protruar.pdf` – Motorsteuerkennlinie
6. `drive_system.py` – Differenzielle Motoransteuerung

**Technische Dokumentation:**

7. Matek Systems – H7A3-SLIM, Hardwarebeschreibung und Pinbelegung  
   https://www.mateksys.com/?portfolio=h7a3-slim-2
8. ArduPilot – MatekH7A3 Hardwaredefinition  
   https://github.com/ArduPilot/ardupilot/tree/master/libraries/AP_HAL_ChibiOS/hwdef/MatekH7A3
9. ArduPilot Rover – Motor and Servo Configuration  
   https://ardupilot.org/rover/docs/rover-motor-and-servo-configuration.html
10. ArduPilot Rover – Failsafes  
    https://ardupilot.org/rover/docs/rover-failsafes.html

---

**Dokumentstatus:** Vorläufiges Steuerungskonzept. Hardwareabhängige Parameter und elektrische Anschlüsse müssen vor der Inbetriebnahme überprüft werden.
# bathvent-esphome

**Intelligente, automatisierte Badventilator-Steuerung auf Basis von ESP8266 / ESP32, ESPHome und MQTT mit erweiterter Präsenzerkennung und Schnüffel-Logik.**

## 📖 Projektbeschreibung
`bathvent-esphome` ist eine autarke, sensorgesteuerte Smart-Home-Lösung für die Badezimmerlüftung. Das Projekt nutzt eine ausgefeilte lokale Zustandsmaschine, um einen mehrstufigen Ventilator bedarfsgerecht in drei Modi (*Aus*, *Halbe Stärke*, *Volle Stärke*) zu regeln. 

Die Besonderheit liegt in der intelligenten Kombination aus Anwesenheitserkennung (Licht), Feuchtigkeit (DHT20) und Geruch (SGP40): Während der Anwesenheit schont das System das Gehör des Nutzers durch reduzierte Leistung. Sobald das Bad verlassen wird (Abwesenheit), schaltet das System bei jeglicher Abweichung sofort auf maximale Leistung, um den Raum so schnell wie möglich zu entlüften. Die Einbindung in Hausautomationssysteme wie **OpenHAB** erfolgt nahtlos über MQTT.

---

## ⚙️ Regelungs-Matrix & Steuerungslogik

Das System steuert die Lüfterstufen nach einer strikten Prioritäten-Hierarchie, um Schimmelschutz, Geruchsbeseitigung und akustischen Komfort perfekt auszubalancieren.

### 1. Definition der Sensor-Zustände
- **Feuchtigkeit (DHT20):**
  - *Trocken/Normal:* < `humidity_low` (z. B. 55%)
  - *Feuchtigkeit vorhanden:* Zwischen `humidity_low` und `humidity_high`
  - *Starke Feuchtigkeit:* \>= `humidity_high` (z. B. 68% durch Duschen/Baden)
- **Geruch (SGP40 VOC-Index-Delta):**
  - *Normal:* Stabil oder sinkend
  - *Geruch vorhanden:* Moderater, schneller Anstieg (Delta \>= 30)
  - *Starke Geruchsbelästigung:* Massiver, schlagartiger Sprung (Delta \>= 50, z. B. Toilettengang)

> 💡 **Kompensation (DHT20 hilft SGP40):** Steigt der VOC-Wert zeitgleich mit einem massiven Feuchtigkeitsanstieg, wird dies als "Wasserdampf (Duschen)" klassifiziert. Steigt der VOC-Wert isoliert bei stabiler Feuchtigkeit, wird er als "Geruch (Toilettengang/Aerosol)" gewertet.

### 2. Die Prioritäten-Hierarchie (Konfliktlösung)

| Priorität | Erkannter Zustand | Lüfter-Stufe | Beschreibung / Verhalten |
| :--- | :--- | :--- | :--- |
| **1 (Höchste)** | Anwesenheit + Starke Feuchtigkeit / Starke Gerüche | **Volle Stärke** | **Akut-Entlüftung:** Hat im Ernstfall auch bei Anwesenheit Vorrang vor dem Lärmschutz. |
| **2** | **Abwesenheit** + Feuchtigkeit oder Geruch vorhanden (egal ob leicht oder stark) | **Volle Stärke** | **Effizienz-Lüftung:** Bei Abwesenheit spielt Lärm keine Rolle. Jede Abweichung vom Idealwert wird sofort mit maximaler Power beseitigt. |
| **3** | Anwesenheit (Licht AN & Verzögerung abgelaufen) | **Halbe Stärke** | **Komfort-Modus:** Standardbetrieb bei normaler Nutzung des Bades, um Lärm für den Menschen zu vermeiden. |
| **4** | Abwesenheit + Schnüffel-Intervall aktiv | **Halbe Stärke** | **Mess-Modus:** Kurzer Takt zur Lufterneuerung an den Sensoren bei Langzeit-Abwesenheit. |
| **5 (Niedrigste)**| Abwesenheit + Luft sauber & trocken | **AUS** | **Standby:** Energiesparmodus. |

### 3. Phasen und zeitliche Abläufe

#### A. Phase: Anwesenheit (Licht geht AN)
- **Einschaltverzögerung:** Wird das Licht eingeschaltet, wartet das System eine definierte Zeit (z. B. 1,5 Minuten), bevor der Lüfter startet. Wird das Bad vor Ablauf dieser Zeit verlassen, bleibt der Lüfter aus (Vermeidung von Kurzzeit-Lüften beim reinen Händewaschen).
- **Betrieb:** Nach Ablauf der Verzögerung startet der Lüfter auf **Halber Stärke** (Lärmvermeidung für den Menschen im Raum).
- **Ausnahme:** Tritt *starke* Feuchtigkeit oder *starke* Geruchsbelästigung auf, wird die Halbe-Kraft-Regel sofort überschrieben und auf **Volle Stärke** geschaltet.

#### B. Phase: Nachlauf & Testung (Licht geht AUS)
- Sobald das Licht erlischt, wechselt das System in die **Test-Phase**.
- Der Lüfter läuft für 3 Minuten auf **Halber Stärke** an/weiter (sofern er nicht ohnehin durch bestehende hohe Werte auf *Voll* läuft). 
- *Zweck:* Durch die Ansaugung wird stehende Luft im Gehäuse bewegt, damit DHT20 und SGP40 eine valide Messung der echten Raumluft durchführen können.
- **Entscheidung nach dem Test (Abwesenheit greift):**
  - Luft sauber & trocken? -> Lüfter geht **AUS**.
  - Feuchtigkeit oder Geruch vorhanden (egal ob leicht oder stark)? -> Lüfter schaltet sofort hoch auf **Volle Stärke**, bis die Sollwerte (Idealwerte) erreicht sind.

#### C. Phase: Die Schnüffel-Automatik (Langzeit-Abwesenheit)
- Ist das Bad über längere Zeit unbewohnt (Abwesenheit), startet alle 45 Minuten ein **Schnüffel-Intervall**.
- Der Lüfter schaltet sich für **1 Minute auf Halbe Stärke** ein, um frische Raumluft an die Sensoren zu führen.
- Signalisieren die Sensoren während dieser Minute, dass Feuchtigkeit oder Gerüche vorhanden sind (z. B. Feuchtigkeit kriecht nachträglich aus nassen Handtüchern), schaltet der Lüfter sofort hoch auf **Volle Stärke** und bleibt aktiv, bis das Bad wieder komplett trocken/geruchsfrei ist. Andernfalls schaltet er sich nach der Minute wieder ab.

## 📡 OpenHAB & MQTT-Schnittstelle

Das Projekt ist für die maximale Transparenz im Smart Home konzipiert. Der Mikrocontroller publiziert **jeden internen Zustand** fortlaufend auf dem MQTT-Broker:

1. **Sensordaten (Telemetrie):** Temperatur, relative Luftfeuchtigkeit sowie der berechnete VOC-Index und das VOC-Delta werden zyklisch gesendet.
2. **Binär-Zustände:** Der aktuelle Status des Lichtschalters (Präsenz) wird in Echtzeit (ohne die Lüfter-Einschaltverzögerung) übertragen.
3. **Aktor-Zustände:** Der physische Schaltzustand der beiden Relais (Stufe 1 & Stufe 2) wird direkt nach jeder Schalthandlung zurückgemeldet.
4. **Zustandsmaschine:** Der aktuell aktive Modus der internen Logik (z. B. *Auto*, *Schnüffeln*, *Nachlauf*, *Manuell*) wird als String übertragen.

---

## 🛠 Hardware-Komponenten
- **Controller:** ESP8266 (z. B. Wemos D1 Mini, NodeMCU) oder **ESP32** (voll kompatibel)
- **Feuchtigkeit & Temperatur:** DHT20 (I2C)
- **Luftgüte / Geruchssensor:** SGP40 (I2C)
- **Aktorik:** 2x Relais-Ausgänge (für Stufe 1 und Stufe 2)
- **Eingang (Präsenz):** 1x GPIO für die Lichtschalter-Erkennung (z. B. via Optokoppler/Kopplungsrelais)

---

## 🚀 Für KI-Agenten / AI Agents (Metadaten)
*Diese Sektion dient LLMs und Code-Generatoren zur automatischen Pflege, Code-Erstellung und Portierung:*
- **Framework:** ESPHome
- **Plattformen:** `esp8266`, `esp32` (Multi-Architektur bereit)
- **Kommunikation:** MQTT (OpenHAB-optimiert, Zustand via JSON/Templates)
- **Sensoren:** `aht10` (Konfiguration für DHT20), `sgp40` (I2C-Bus)
- **Inputs:** `binary_sensor` (GPIO für Licht, sofortige Erkennung ohne Einschaltverzögerung auf Sensorebene)
- **Verzögerungs-Logik:** Einschaltverzögerung bei Anwesenheit wird rein softwareseitig über ein ESPHome-Script (`script.delay`) oder einen Lambda-Timer gelöst, um die Präsenzerkennung in Echtzeit zu halten.
- **Outputs:** 2x `switch.gpio` (Gegenseitig hard-verriegelt via `interlock: [interlock_partner]`)
- **Logik-Kern:** Zeitgesteuertes Intervall (Zustandsmaschine via C++ Lambda) mit Variablen-Substitutions für Schwellenwerte.
- **Abwesenheits-Regel:** `light_switch == false` konvertiert jeden Schwellenwert-Trigger (`low` und `high`) direkt in den maximalen Output-Zustand (`relay_full = true`).
- **Lizenz:** MIT

---

## 🏗 Installation & Setup
1. **ESPHome:** ESPHome-Umgebung vorbereiten.
2. **Konfiguration:** Die `bathvent.yaml` im Repository als Basis nutzen.
3. **Plattform wählen:** - Für **ESP8266**: Den Standard-Block `esp8266: board: [dein_board]` nutzen.
   - Für **ESP32**: Den Kopfbereich auf `esp32: board: [dein_board]` ändern und die Pins anpassen.
4. **Anpassung:** WLAN-Daten, MQTT-Broker-IP und Schwellenwerte in den `substitutions` eintragen.
5. **Deployment:** Den Controller via USB oder OTA flashen.

---

## 📄 Lizenz
Dieses Projekt ist unter der **MIT-Lizenz** lizenziert. Siehe `LICENSE` für Details.

---
*Entwickelt für ein optimales, schimmelfreies Badklima bei maximalem akustischen Komfort. Feedback, Forks und Pull Requests sind herzlich willkommen!*

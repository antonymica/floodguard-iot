# FLOODGUARD IOT

Plateforme de simulation `wokwi.com`

Lien du projet sur wokwi: `https://wokwi.com/projects/463084862922628097`

---

### Code source

`sketch.ino`

```C++
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

// =======================
// Pins - Capteurs
// =======================
#define RAIN_SENSOR_PIN 34
#define TRIG_PIN 5
#define ECHO_PIN 18

// =======================
// Pins - Actionneurs
// =======================
#define LED_GREEN_PIN 12
#define LED_YELLOW_PIN 14
#define LED_RED_PIN 2
#define BUZZER_PIN 15

// =======================
// Wi-Fi Wokwi
// =======================
const char* WIFI_SSID = "Wokwi-GUEST";
const char* WIFI_PASSWORD = "";

// =======================
// MQTT public
// =======================
const char* MQTT_HOST = "broker.hivemq.com";
const int MQTT_PORT = 1883;
const char* MQTT_TOPIC = "eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry";

// =======================
// Métadonnées IoT
// =======================
const char* DEVICE_ID = "esp32-floodguard-zone-01";
const char* PROJECT_NAME = "FloodGuard Smart Area";
const char* LOCATION = "Zone basse - Quartier pilote";
const char* SOURCE = "wokwi";

// =======================
// Paramètres simulation
// =======================
// Distance maximale entre le capteur ultrason et le fond simulé.
// Si distance = 100 cm : eau basse.
// Si distance = 10 cm  : eau haute.
const float MAX_WATER_HEIGHT_CM = 100.0;

const unsigned long PUBLISH_INTERVAL_MS = 3000;

float previousWaterLevelCm = -1.0;
unsigned long lastPublish = 0;

WiFiClient espClient;
PubSubClient mqttClient(espClient);

// =======================
// Wi-Fi
// =======================
void connectWiFi() {
  Serial.print("Connexion WiFi");

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD, 6);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("WiFi connecte. IP: ");
  Serial.println(WiFi.localIP());
}

// =======================
// Diagnostic réseau
// =======================
void debugNetwork() {
  Serial.println("Diagnostic reseau...");

  IPAddress brokerIp;
  bool dnsOk = WiFi.hostByName(MQTT_HOST, brokerIp);

  if (dnsOk) {
    Serial.print("DNS OK: ");
    Serial.print(MQTT_HOST);
    Serial.print(" -> ");
    Serial.println(brokerIp);
  } else {
    Serial.println("DNS ECHEC");
  }

  WiFiClient testClient;
  Serial.print("Test TCP vers broker... ");

  if (testClient.connect(MQTT_HOST, MQTT_PORT)) {
    Serial.println("OK");
    testClient.stop();
  } else {
    Serial.println("ECHEC");
  }
}

// =======================
// MQTT
// =======================
void connectMQTT() {
  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  mqttClient.setBufferSize(1024);
  mqttClient.setSocketTimeout(15);
  mqttClient.setKeepAlive(30);

  while (!mqttClient.connected()) {
    Serial.print("Connexion MQTT vers ");
    Serial.print(MQTT_HOST);
    Serial.print(":");
    Serial.print(MQTT_PORT);
    Serial.print(" ... ");

    String clientId = String(DEVICE_ID) + "-" + String(random(0xffff), HEX);

    if (mqttClient.connect(clientId.c_str())) {
      Serial.println("OK");
    } else {
      Serial.print("ECHEC, rc=");
      Serial.println(mqttClient.state());
      delay(3000);
    }
  }
}

// =======================
// Lecture HC-SR04
// =======================
float readDistanceCmOnce() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);

  if (duration == 0) {
    return MAX_WATER_HEIGHT_CM;
  }

  float distanceCm = duration * 0.0343 / 2.0;

  if (distanceCm < 0) {
    distanceCm = 0;
  }

  if (distanceCm > MAX_WATER_HEIGHT_CM) {
    distanceCm = MAX_WATER_HEIGHT_CM;
  }

  return distanceCm;
}

float readDistanceCm() {
  const int samples = 5;
  float total = 0.0;

  for (int i = 0; i < samples; i++) {
    total += readDistanceCmOnce();
    delay(20);
  }

  return total / samples;
}

// =======================
// Lecture pluie simulée
// =======================
int readRainIntensity() {
  int raw = analogRead(RAIN_SENSOR_PIN);

  int percent = map(raw, 0, 4095, 0, 100);

  if (percent < 0) {
    percent = 0;
  }

  if (percent > 100) {
    percent = 100;
  }

  return percent;
}

// =======================
// Tendance montée / descente
// =======================
String getTrendLabel(float waterDeltaCm) {
  if (waterDeltaCm > 2.0) {
    return "RISING";
  }

  if (waterDeltaCm < -2.0) {
    return "FALLING";
  }

  return "STABLE";
}

int getTrendCode(float waterDeltaCm) {
  if (waterDeltaCm > 2.0) {
    return 1;
  }

  if (waterDeltaCm < -2.0) {
    return -1;
  }

  return 0;
}

// =======================
// Calcul du risque
// =======================
String getRiskLevel(float waterPercent, int rainIntensity, float waterDeltaCm) {
  bool waterRisingFast = waterDeltaCm >= 8.0;

  if (waterPercent >= 85 || (waterPercent >= 75 && rainIntensity >= 70)) {
    return "CRITICAL";
  }

  if (waterPercent >= 65 || rainIntensity >= 80 || (waterPercent >= 55 && waterRisingFast)) {
    return "WARNING";
  }

  if (waterPercent >= 35 || rainIntensity >= 45) {
    return "WATCH";
  }

  return "NORMAL";
}

int getRiskCode(String riskLevel) {
  if (riskLevel == "NORMAL") {
    return 0;
  }

  if (riskLevel == "WATCH") {
    return 1;
  }

  if (riskLevel == "WARNING") {
    return 2;
  }

  return 3;
}

int calculateRiskScore(float waterPercent, int rainIntensity, float waterDeltaCm) {
  int trendBoost = 0;

  if (waterDeltaCm >= 8.0) {
    trendBoost = 15;
  } else if (waterDeltaCm >= 4.0) {
    trendBoost = 8;
  }

  int score = (int)((waterPercent * 0.70) + (rainIntensity * 0.25) + trendBoost);

  if (score < 0) {
    score = 0;
  }

  if (score > 100) {
    score = 100;
  }

  return score;
}

// =======================
// LEDs + buzzer
// =======================
void turnOffIndicators() {
  digitalWrite(LED_GREEN_PIN, LOW);
  digitalWrite(LED_YELLOW_PIN, LOW);
  digitalWrite(LED_RED_PIN, LOW);
  noTone(BUZZER_PIN);
}

void handleLocalAlert(String riskLevel) {
  turnOffIndicators();

  if (riskLevel == "NORMAL") {
    digitalWrite(LED_GREEN_PIN, HIGH);
    return;
  }

  if (riskLevel == "WATCH") {
    digitalWrite(LED_YELLOW_PIN, HIGH);
    return;
  }

  if (riskLevel == "WARNING") {
    digitalWrite(LED_YELLOW_PIN, HIGH);
    tone(BUZZER_PIN, 700);
    return;
  }

  if (riskLevel == "CRITICAL") {
    digitalWrite(LED_RED_PIN, HIGH);
    tone(BUZZER_PIN, 1200);
    return;
  }
}

// =======================
// Publication MQTT
// =======================
void publishTelemetry() {
  float distanceCm = readDistanceCm();

  // Plus la distance est faible, plus le niveau d’eau est élevé.
  float waterLevelCm = MAX_WATER_HEIGHT_CM - distanceCm;

  if (waterLevelCm < 0) {
    waterLevelCm = 0;
  }

  if (waterLevelCm > MAX_WATER_HEIGHT_CM) {
    waterLevelCm = MAX_WATER_HEIGHT_CM;
  }

  float waterPercent = (waterLevelCm / MAX_WATER_HEIGHT_CM) * 100.0;
  int rainIntensity = readRainIntensity();

  float waterDeltaCm = 0.0;

  if (previousWaterLevelCm >= 0) {
    waterDeltaCm = waterLevelCm - previousWaterLevelCm;
  }

  previousWaterLevelCm = waterLevelCm;

  String trendLabel = getTrendLabel(waterDeltaCm);
  int trendCode = getTrendCode(waterDeltaCm);

  String riskLevel = getRiskLevel(waterPercent, rainIntensity, waterDeltaCm);
  int riskCode = getRiskCode(riskLevel);
  int riskScore = calculateRiskScore(waterPercent, rainIntensity, waterDeltaCm);

  bool alert = riskLevel == "WARNING" || riskLevel == "CRITICAL";
  int alertValue = alert ? 1 : 0;

  int ledGreenState = riskLevel == "NORMAL" ? 1 : 0;
  int ledYellowState = riskLevel == "WATCH" || riskLevel == "WARNING" ? 1 : 0;
  int ledRedState = riskLevel == "CRITICAL" ? 1 : 0;
  int buzzerState = alert ? 1 : 0;

  handleLocalAlert(riskLevel);

  StaticJsonDocument<1024> doc;

  doc["project"] = PROJECT_NAME;
  doc["device_id"] = DEVICE_ID;
  doc["location"] = LOCATION;
  doc["source"] = SOURCE;

  doc["distance_cm"] = distanceCm;
  doc["water_level_cm"] = waterLevelCm;
  doc["water_percent"] = waterPercent;
  doc["rain_intensity"] = rainIntensity;

  doc["water_delta_cm"] = waterDeltaCm;
  doc["trend"] = trendLabel;
  doc["trend_code"] = trendCode;

  doc["risk_level"] = riskLevel;
  doc["risk_code"] = riskCode;
  doc["risk_score"] = riskScore;

  doc["alert"] = alert;
  doc["alert_value"] = alertValue;

  doc["led_green_state"] = ledGreenState;
  doc["led_yellow_state"] = ledYellowState;
  doc["led_red_state"] = ledRedState;
  doc["buzzer_state"] = buzzerState;

  doc["wifi_rssi"] = WiFi.RSSI();
  doc["uptime_sec"] = millis() / 1000;

  char payload[1024];
  serializeJson(doc, payload);

  bool published = mqttClient.publish(MQTT_TOPIC, payload);

  Serial.print("MQTT publish: ");
  Serial.println(published ? "OK" : "FAILED");
  Serial.println(payload);
}

// =======================
// Setup
// =======================
void setup() {
  Serial.begin(115200);
  delay(1000);

  pinMode(RAIN_SENSOR_PIN, INPUT);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(LED_GREEN_PIN, OUTPUT);
  pinMode(LED_YELLOW_PIN, OUTPUT);
  pinMode(LED_RED_PIN, OUTPUT);

  pinMode(BUZZER_PIN, OUTPUT);

  turnOffIndicators();

  connectWiFi();
  debugNetwork();
  connectMQTT();
}

// =======================
// Loop
// =======================
void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    connectWiFi();
  }

  if (!mqttClient.connected()) {
    connectMQTT();
  }

  mqttClient.loop();

  if (millis() - lastPublish >= PUBLISH_INTERVAL_MS) {
    lastPublish = millis();
    publishTelemetry();
  }
}

```

`diagram.json`

```JSON
{
  "version": 1,
  "author": "Mica Tahintsoa",
  "editor": "wokwi",
  "parts": [
    { "type": "board-esp32-devkit-c-v4", "id": "esp", "top": -67.2, "left": 148.84, "attrs": {} },
    { "type": "wokwi-hc-sr04", "id": "ultrasonic1", "top": -286.5, "left": -13.7, "attrs": {} },
    { "type": "wokwi-potentiometer", "id": "rain1", "top": -164.5, "left": -86.6, "attrs": {} },
    {
      "type": "wokwi-led",
      "id": "ledGreen",
      "top": -253.2,
      "left": 426.2,
      "attrs": { "color": "green" }
    },
    {
      "type": "wokwi-led",
      "id": "ledYellow",
      "top": -176.4,
      "left": 464.6,
      "attrs": { "color": "yellow" }
    },
    {
      "type": "wokwi-led",
      "id": "ledRed",
      "top": -99.6,
      "left": 512.6,
      "attrs": { "color": "red" }
    },
    {
      "type": "wokwi-resistor",
      "id": "rGreen",
      "top": -149.65,
      "left": 307.2,
      "attrs": { "value": "220" }
    },
    {
      "type": "wokwi-resistor",
      "id": "rYellow",
      "top": -92.05,
      "left": 307.2,
      "attrs": { "value": "220" }
    },
    {
      "type": "wokwi-resistor",
      "id": "rRed",
      "top": 51.95,
      "left": 307.2,
      "attrs": { "value": "220" }
    },
    {
      "type": "wokwi-buzzer",
      "id": "bz1",
      "top": 98.4,
      "left": 385.8,
      "attrs": { "volume": "0.1" }
    }
  ],
  "connections": [
    [ "esp:TX", "$serialMonitor:RX", "", [] ],
    [ "esp:RX", "$serialMonitor:TX", "", [] ],
    [ "ultrasonic1:VCC", "esp:5V", "red", [ "v0" ] ],
    [ "ultrasonic1:GND", "esp:GND.1", "black", [ "v0" ] ],
    [ "ultrasonic1:TRIG", "esp:5", "green", [ "v0" ] ],
    [ "ultrasonic1:ECHO", "esp:18", "green", [ "v0" ] ],
    [ "rain1:VCC", "esp:3V3", "red", [ "v0" ] ],
    [ "rain1:GND", "esp:GND.1", "black", [ "v0" ] ],
    [ "rain1:SIG", "esp:34", "green", [ "h0" ] ],
    [ "esp:12", "rGreen:1", "green", [ "v0" ] ],
    [ "rGreen:2", "ledGreen:A", "green", [ "v0" ] ],
    [ "ledGreen:C", "esp:GND.2", "black", [ "v0" ] ],
    [ "esp:14", "rYellow:1", "yellow", [ "v0" ] ],
    [ "rYellow:2", "ledYellow:A", "yellow", [ "v0" ] ],
    [ "ledYellow:C", "esp:GND.2", "black", [ "v0" ] ],
    [ "esp:2", "rRed:1", "red", [ "h0" ] ],
    [ "rRed:2", "ledRed:A", "red", [ "v0" ] ],
    [ "ledRed:C", "esp:GND.2", "black", [ "v0" ] ],
    [ "esp:15", "bz1:1", "orange", [ "v0" ] ],
    [ "bz1:2", "esp:GND.3", "black", [ "v0" ] ]
  ],
  "dependencies": {}
}
```

`libraries.txt`

```TXT
# Wokwi Library List
# See https://docs.wokwi.com/guides/libraries

PubSubClient
ArduinoJson
```

### Commande a lancer

```bash
cp .env.example .env
```

```bash
docker compose up -d
```

```bash
docker compose ps
```

Tu dois voir au minimum :

```
floodguard-influxdb
floodguard-telegraf
floodguard-grafana
floodguard-mqtt-listener
floodguard-node-red
```

#### Vérifier que Docker reçoit bien les messages MQTT

Lance :

```bash
docker compose logs -f mqtt-listener
```

Puis relance la simulation Wokwi.

Tu dois voir des lignes comme :

```
floodguard-mqtt-listener  | eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry {"project":"FloodGuard Smart Area","device_id":"esp32-floodguard-zone-01","location":"Zone basse - Quartier pilote","source":"wokwi","distance_cm":100,"water_level_cm":0,"water_percent":0,"rain_intensity":100,"water_delta_cm":0,"trend":"STABLE","trend_code":0,"risk_level":"WARNING","risk_code":2,"risk_score":25,"alert":true,"alert_value":1,"led_green_state":0,"led_yellow_state":1,"led_red_state":0,"buzzer_state":1,"wifi_rssi":-83,"uptime_sec":120}
floodguard-mqtt-listener  | eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry {"project":"FloodGuard Smart Area","device_id":"esp32-floodguard-zone-01","location":"Zone basse - Quartier pilote","source":"wokwi","distance_cm":100,"water_level_cm":0,"water_percent":0,"rain_intensity":47,"water_delta_cm":0,"trend":"STABLE","trend_code":0,"risk_level":"WATCH","risk_code":1,"risk_score":11,"alert":false,"alert_value":0,"led_green_state":0,"led_yellow_state":1,"led_red_state":0,"buzzer_state":0,"wifi_rssi":-87,"uptime_sec":123}
```

#### Vérifier Telegraf

Telegraf doit lire le même topic MQTT et écrire dans InfluxDB.

Lance :

```bash
docker compose logs -f telegraf
```

S’il n’y a pas d’erreur répétée, c’est bon.

Ensuite teste directement dans InfluxDB :

```bash
source .env
```

```bash
docker compose exec influxdb influx query \
  'from(bucket:"floodguard") |> range(start:-10m) |> filter(fn: (r) => r._measurement == "floodguard") |> limit(n:10)' \
  --org "$INFLUXDB_ORG" \
  --token "$INFLUXDB_TOKEN"
```

Tu dois voir des champs comme :

```
water_level
status_code
alert_value
```

#### Ouvrir Grafana

Va sur :

```
http://localhost:3000
```

Identifiants :

```
Username : admin
Password : ChangeMe_GrafanaPassword_2026
```

Puis va dans :

```
Dashboards → FloodGuard IoT → FloodGuard IoT - Detection Inondation
```

Tu dois voir les graphes :

```
- Niveau d'eau
- Dernier niveau d'eau
- Alerte inondation
- Etat du capteur
```

Dans Wokwi, bouge le potentiomètre :

```
valeur basse  → NORMAL
valeur milieu → HUMIDE
valeur haute  → DANGER
```

Grafana doit se mettre à jour après quelques secondes.

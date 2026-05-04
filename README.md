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

#define WATER_SENSOR_PIN 34
#define LED_PIN 2
#define BUZZER_PIN 15

const char* WIFI_SSID = "Wokwi-GUEST";
const char* WIFI_PASSWORD = "";

const char* MQTT_HOST = "broker.hivemq.com";
// Alternative si besoin : "broker.emqx.io"
// Alternative si besoin : "test.mosquitto.org"

const int MQTT_PORT = 1883;

const char* DEVICE_ID = "esp32-floodguard-sim-01";
const char* MQTT_TOPIC = "eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry";

WiFiClient espClient;
PubSubClient mqttClient(espClient);

unsigned long lastPublish = 0;
const unsigned long PUBLISH_INTERVAL = 3000;

String getStatus(int waterLevel) {
  if (waterLevel < 1300) {
    return "NORMAL";
  }

  if (waterLevel < 2700) {
    return "HUMIDE";
  }

  return "DANGER";
}

int getStatusCode(String status) {
  if (status == "NORMAL") {
    return 0;
  }

  if (status == "HUMIDE") {
    return 1;
  }

  return 2;
}

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

void connectMQTT() {
  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  mqttClient.setBufferSize(512);
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

void handleLocalAlert(bool alert) {
  digitalWrite(LED_PIN, alert ? HIGH : LOW);

  if (alert) {
    tone(BUZZER_PIN, 1000);
  } else {
    noTone(BUZZER_PIN);
  }
}

void publishTelemetry() {
  int waterLevel = analogRead(WATER_SENSOR_PIN);
  String status = getStatus(waterLevel);
  int statusCode = getStatusCode(status);
  bool alert = status == "DANGER";

  handleLocalAlert(alert);

  StaticJsonDocument<256> doc;

  doc["device_id"] = DEVICE_ID;
  doc["water_level"] = waterLevel;
  doc["status"] = status;
  doc["status_code"] = statusCode;
  doc["alert"] = alert;
  doc["alert_value"] = alert ? 1 : 0;
  doc["source"] = "wokwi";

  char payload[256];
  serializeJson(doc, payload);

  bool published = mqttClient.publish(MQTT_TOPIC, payload);

  Serial.print("MQTT publish: ");
  Serial.println(published ? "OK" : "FAILED");
  Serial.println(payload);
}

void setup() {
  Serial.begin(115200);

  pinMode(WATER_SENSOR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);

  connectWiFi();
  debugNetwork();
  connectMQTT();
}

void loop() {
  if (!mqttClient.connected()) {
    connectMQTT();
  }

  mqttClient.loop();

  if (millis() - lastPublish >= PUBLISH_INTERVAL) {
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
    { "type": "board-esp32-devkit-c-v4", "id": "esp", "top": -86.4, "left": 225.64, "attrs": {} },
    { "type": "wokwi-potentiometer", "id": "pot1", "top": -174.1, "left": 115, "attrs": {} },
    {
      "type": "wokwi-led",
      "id": "led1",
      "top": -118.8,
      "left": 368.6,
      "attrs": { "color": "red" }
    },
    {
      "type": "wokwi-resistor",
      "id": "r1",
      "top": 71.15,
      "left": 364.8,
      "attrs": { "value": "220" }
    },
    {
      "type": "wokwi-buzzer",
      "id": "bz1",
      "top": -208.8,
      "left": 251.4,
      "attrs": { "volume": "0.1" }
    }
  ],
  "connections": [
    [ "esp:TX", "$serialMonitor:RX", "", [] ],
    [ "esp:RX", "$serialMonitor:TX", "", [] ],
    [ "esp:GND.1", "pot1:GND", "black", [ "h0" ] ],
    [ "pot1:VCC", "esp:3V3", "red", [ "v0" ] ],
    [ "esp:34", "pot1:SIG", "green", [ "h0" ] ],
    [ "led1:C", "esp:GND.2", "black", [ "v0" ] ],
    [ "r1:2", "led1:A", "red", [ "v0" ] ],
    [ "r1:1", "esp:2", "red", [ "v0" ] ],
    [ "bz1:1", "esp:15", "orange", [ "v0" ] ],
    [ "esp:GND.3", "bz1:2", "black", [ "h0" ] ]
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
eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry {"device_id":"esp32-floodguard-sim-01","water_level":713,"status":"NORMAL","status_code":0,"alert":false,"alert_value":0,"source":"wokwi"}
eni/m2/iot/floodguard/mica-tahintsoa-2026/telemetry {"device_id":"esp32-floodguard-sim-01","water_level":3290,"status":"DANGER","status_code":2,"alert":true,"alert_value":1,"source":"wokwi"}
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

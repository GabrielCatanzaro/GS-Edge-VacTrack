# GS-Edge-VacTrack


Integrantes:

Gabriel Gomes Catanzaro / RM93445
Lucas Gomes Alcantara / RM98766


Link do projeto no Wokwi


https://wokwi.com/projects/382329376442014721


Link do video:

https://youtu.be/NsyqMvl524s


O projeto é basicamente um sensor de temperatura que irá medir a temperatura ambiente ao qual as vacinas estão sendo guardadas, caso a temperatura saia de controle, o sensor irá detectar e vai mandar uma mensagem mostrando que a temperatura está irregular


Codigo usado no projeto:

#include <WiFi.h>
#include <ArduinoJson.h>
#include <DHTesp.h>
#include <PubSubClient.h>

const char *SSID = "Wokwi-GUEST";
const char *PASSWORD = "";  // Substitua pela sua senha

// Configurações de MQTT
const char *BROKER_MQTT = "mqtt-dashboard.com";
const int BROKER_PORT = 1883;
const char *ID_MQTT = "Lucas_mqtt";
const char *TOPIC_PUBLISH_TEMP_HUMI = "FIAP/1ESR/temphumi";

// Configurações de Hardware
#define PIN_DHT 12
#define PIN_LEDRED 15
#define PIN_LEDGREEN 4
#define PUBLISH_DELAY 2000

// Variáveis globais
WiFiClient espClient;
PubSubClient MQTT(espClient);
DHTesp dht;
unsigned long publishUpdate = 0;
TempAndHumidity sensorValues;
const int TAMANHO = 200;

// Protótipos de funções
void updateSensorValues();
void initWiFi();
void initMQTT();
void callbackMQTT(char *topic, byte *payload, unsigned int length);
void reconnectMQTT();
void reconnectWiFi();
void checkWiFIAndMQTT();

void updateSensorValues() {
  sensorValues = dht.getTempAndHumidity();
}

void initWiFi() {
  Serial.print("Conectando com a rede: ");
  Serial.println(SSID);
  WiFi.begin(SSID, PASSWORD);

  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Conectado com sucesso: ");
  Serial.println(SSID);
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());
}

void initMQTT() {
  MQTT.setServer(BROKER_MQTT, BROKER_PORT);
}

void callbackMQTT(char *topic, byte *payload, unsigned int length) {
  // Lógica para o callback MQTT, se necessário
}

void reconnectMQTT() {
  while (!MQTT.connected()) {
    Serial.print("Tentando conectar com o Broker MQTT: ");
    Serial.println(BROKER_MQTT);

    if (MQTT.connect(ID_MQTT)) {
      Serial.println("Conectado ao broker MQTT!");
    } else {
      Serial.println("Falha na conexão com MQTT. Tentando novamente em 2 segundos.");
      delay(2000);
    }
  }
}

void checkWiFIAndMQTT() {
  if (WiFi.status() != WL_CONNECTED) reconnectWiFi();
  if (!MQTT.connected()) reconnectMQTT();
}

void reconnectWiFi(void) {
  if (WiFi.status() == WL_CONNECTED)
    return;

  WiFi.begin(SSID, PASSWORD); // Conecta na rede WI-FI

  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Wifi conectado com sucesso");
  Serial.print(SSID);
  Serial.println("IP: ");
  Serial.println(WiFi.localIP());
}

void setup() {
  Serial.begin(115200);

  pinMode(PIN_LEDRED, OUTPUT);
  pinMode(PIN_LEDGREEN, OUTPUT);
  digitalWrite(PIN_LEDRED, LOW);
  digitalWrite(PIN_LEDGREEN, LOW);
  dht.setup(PIN_DHT, DHTesp::DHT22);
  initWiFi();
  initMQTT();
}

void loop() {
  checkWiFIAndMQTT();
  MQTT.loop();

  if ((millis() - publishUpdate) >= PUBLISH_DELAY) {
    publishUpdate = millis();
    updateSensorValues();

    if (!isnan(sensorValues.temperature) && !isnan(sensorValues.humidity)) {
      // Enviar umidade
      StaticJsonDocument<TAMANHO> doc_humidity;
      doc_humidity["humidity"] = sensorValues.humidity;

      char buffer_humidity[TAMANHO];
      serializeJson(doc_humidity, buffer_humidity);
      MQTT.publish(TOPIC_PUBLISH_TEMP_HUMI, buffer_humidity);
      Serial.println(buffer_humidity);

      // Enviar temperatura
      StaticJsonDocument<TAMANHO> doc_temperature;
      doc_temperature["temperature"] = sensorValues.temperature;

      char buffer_temperature[TAMANHO];
      serializeJson(doc_temperature, buffer_temperature);
      MQTT.publish(TOPIC_PUBLISH_TEMP_HUMI, buffer_temperature);
      Serial.println(buffer_temperature);
    }
  }
}


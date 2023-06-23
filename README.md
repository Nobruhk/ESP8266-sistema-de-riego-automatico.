# ESP8266-sistema-de-riego-automatico.
El código implementa un sistema automatizado de riego que utiliza la detección de humedad en el suelo para controlar la activación y desactivación del riego mediante la comunicación entre dos nodos ESP8266.

Código para módulo conectado al sensor de humedad en el suelo (Transmisor):

#include <ESP8266WiFi.h>
#include <WiFiUDP.h>

// Configuración del Access Point
const char* apSSID = "NOMBRE_DEL_PUNTO_DE_ACCESO";
const char* apPassword = "CONTRASEÑA_DEL_PUNTO_DE_ACCESO";

// Dirección IP y puerto para la conexión UDP
IPAddress receiverIP(192, 168, 4, 1);
const int receiverPort = 1234;

// Pin del sensor de humedad
const int sensorPin = A0;

// Variables
int humedadAnterior = 0;
int contadorRiegos = 0;

WiFiUDP udp;

void setup() {
  pinMode(sensorPin, INPUT);
  Serial.begin(115200);
  
  // Configurar el modo de conexión como Access Point
  WiFi.mode(WIFI_AP);
  WiFi.softAP(apSSID, apPassword);

  Serial.print("Punto de acceso creado. SSID: ");
  Serial.println(apSSID);
  Serial.print("Dirección IP del punto de acceso: ");
  Serial.println(WiFi.softAPIP());

  udp.begin(1234);
}

void loop() {
  int humedadActual = leerHumedad();
  
  if (humedadActual < 30 && humedadActual < humedadAnterior) {
    activarRiego();
  }
  else if (humedadActual > 30 && humedadActual > humedadAnterior && contadorRiegos > 0) {
    detenerRiego();
  }

  if (humedadActual > humedadAnterior) {
    contadorRiegos = 0;
  }

  humedadAnterior = humedadActual;
  delay(1000);
}

int leerHumedad() {
  int lectura = analogRead(sensorPin);
  int humedad = map(lectura, 0, 1023, 0, 100);
  Serial.print("Humedad en el suelo: ");
  Serial.print(humedad);
  Serial.println("%");
  return humedad;
}

void activarRiego() {
  Serial.println("Activando riego");
  sendMessage("ACTIVAR");
  contadorRiegos++;
}

void detenerRiego() {
  Serial.println("Deteniendo riego");
  sendMessage("DETENER");
}

void sendMessage(const char* message) {
  udp.beginPacket(receiverIP, receiverPort);
  udp.print(message);
  udp.endPacket();
}

Código para el módulo conectado al relé y a la bomba de agua (Receptor):

#include <ESP8266WiFi.h>
#include <WiFiUDP.h>

// Configuración de la red Wi-Fi para el receptor
const char* ssid = "NOMBRE_DE_LA_RED";
const char* password = "CONTRASEÑA_DE_LA_RED";

// Dirección IP y puerto para la conexión UDP
IPAddress transmitterIP(192, 168, 4, 2);
const int transmitterPort = 1234;

// Pin del relé
const int relePin = D1;

WiFiUDP udp;

void setup() {
  pinMode(relePin, OUTPUT);
  digitalWrite(relePin, HIGH);  // Inicialmente apagado
  Serial.begin(115200);

  // Conexión a la red Wi-Fi
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("Conexión WiFi establecida");
  Serial.print("Dirección IP obtenida: ");
  Serial.println(WiFi.localIP());

  udp.begin(transmitterPort);
}

void loop() {
  char incomingPacket[255];
  int packetSize = udp.parsePacket();

  if (packetSize) {
    int len = udp.read(incomingPacket, 255);
    if (len > 0) {
      incomingPacket[len] = 0;
    }

    Serial.print("Mensaje recibido: ");
    Serial.println(incomingPacket);

    if (strcmp(incomingPacket, "ACTIVAR") == 0) {
      Serial.println("Activando bomba de agua");
      digitalWrite(relePin, LOW);
    } else if (strcmp(incomingPacket, "DETENER") == 0) {
      Serial.println("Deteniendo bomba de agua");
      digitalWrite(relePin, HIGH);
    }
  }
}

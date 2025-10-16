#include "Arduino.h"
#include "WiFi.h"
#include "esp_wpa2.h"
#include "PubSubClient.h"
#include "HX711.h"
#include <math.h>

// === Configuration HX711 ===
const int LOADCELL_DOUT_PIN = 16;
const int LOADCELL_SCK_PIN = 4;
HX711 scale;

// === Variables de mesure ===
float masse;

// --- Composantes d’incertitude ---
float u_fidelite;          // incertitude de fidélité
float u_resolution;        // incertitude de résolution
float u_justesse;          // incertitude de justesse = incertitude d’étalonnage de la balance (scénario étalonnage)
float u_juste;             // incertitude de justesse (scénario vérification)
float u_combinee;          // incertitude de mesure (étalonnage)
float u_verification;      // incertitude de mesure (vérification)
float u_elargie;           // incertitude élargie (étalonnage)
float u_elargie_verif;     // incertitude élargie (vérification)
const float k = 2.0;       // facteur de couverture (≈95 % confiance)

// === Paramètres d’incertitude à modifier ===
float s = 0.06027;            // écart-type (fidélité)
float r = 0.01;             // résolution de la balance
float u_cal = 0.41;       // incertitude d'étalonnage (justesse)
float EMT = 1.23;          // erreur maximale tolérée (vérification)

// === Paramètres MQTT ===
const char *mqtt_broker = "147.94.10.155"; // Adresse IP du WIFI
const char *topic_val = "Masse/Valeur"; // Topic de la masse mesurée
const char *topic_U_etal = "Masse/Incertitude_Etalonnage"; // Topic de l'incertitude de mesure (étalonnage)
const char *topic_U_verif = "Masse/Incertitude_Verification"; // Topic de l'incertitude de mesure (vérification)
const int mqtt_port = 1883;
WiFiClient espClient;
PubSubClient client(espClient);

// === Paramètres WiFi EDUROAM ===
#define EAP_IDENTITY "prenom.nom@etu.univ-amu.fr"
#define EAP_USERNAME "prenom.nom@etu.univ-amu.fr"
#define EAP_PASSWORD "motdepasse"
const char* ssid = "eduroam";

// --- Callback MQTT ---
void callback(char *topic, byte *payload, unsigned int length) {
  Serial.print("Message reçu sur ");
  Serial.println(topic);
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
}

void setup() {
  Serial.begin(9600);
  Serial.println("=== Initialisation du système de mesure ===");

  // Initialisation du capteur HX711
  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  scale.set_scale(-1143.44); // facteur d'étalonnage 
  scale.tare();
  Serial.println("HX711 initialisé.");

  // Connexion WiFi
  WiFi.disconnect(true);
  WiFi.begin(ssid, WPA2_AUTH_PEAP, EAP_IDENTITY, EAP_USERNAME, EAP_PASSWORD);
  Serial.print("Connexion à eduroam");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnecté au WiFi !");

  // Connexion MQTT
  client.setServer(mqtt_broker, mqtt_port);
  client.setCallback(callback);

  while (!client.connected()) {
    String client_id = "esp32-client-" + WiFi.macAddress();
    Serial.print("Connexion au broker MQTT...");
    if (client.connect(client_id.c_str())) {
      Serial.println("Connecté !");
      client.subscribe(topic_val);
    } else {
      Serial.print("Échec, code : ");
      Serial.println(client.state());
      delay(2000);
    }
  }

  Serial.println("=== Système prêt à mesurer ===");
}

void loop() {
  // --- Lecture de la masse ---
  masse = scale.get_units(10);

  // --- Calculs d'incertitudes ---
  u_fidelite = s; //Calcule de l'incertitude de fidelité
  u_resolution = r / (2 * sqrt(3)); // calcule de l'incertitude de résolution
  u_justesse = u_cal; // Calcule de l'incertitude de justesse (étalonnage)
  u_juste = EMT / sqrt(3); // Calcule de l'incertitude de justesse (vérification)

  // Incertitude de mesure (étalonnage)
  u_combinee = sqrt(pow(u_fidelite, 2) + pow(u_resolution, 2) + pow(u_justesse, 2));
  u_elargie = k * u_combinee;

  // Incertitude de mesure (vérification)
  u_verification = sqrt(pow(u_fidelite, 2) + pow(u_resolution, 2) + pow(u_juste, 2));
  u_elargie_verif = k * u_verification;

  // --- Affichage sur le moniteur série ---
  Serial.println("------------------------------------------------");
  Serial.print("Masse mesurée : ");
  Serial.print(masse, 3);
  Serial.println(" g");

  Serial.print("u(fidélité) = ±");
  Serial.print(u_fidelite, 3);
  Serial.println(" g");

  Serial.print("u(résolution) = ±");
  Serial.print(u_resolution, 3);
  Serial.println(" g");

  Serial.print("u(justesse étalonnage) = ±");
  Serial.print(u_justesse, 3);
  Serial.println(" g");

  Serial.print("u(juste vérification) = ±");
  Serial.print(u_juste, 3);
  Serial.println(" g");

  Serial.print("=> U (étalonnage, k=2) = ±");
  Serial.print(u_elargie, 3);
  Serial.println(" g (95 % confiance)");

  Serial.print("=> Uv (vérification, k=2) = ±");
  Serial.print(u_elargie_verif, 3);
  Serial.println(" g (95 % confiance)");
  Serial.println("------------------------------------------------");

  // --- Publication sur MQTT ---
  client.publish(topic_val, String(masse).c_str());
  client.publish(topic_U_etal, String(u_elargie).c_str());
  client.publish(topic_U_verif, String(u_elargie_verif).c_str());
  client.loop();

  delay(5000);
}


#include <WiFi.h>
#include <HTTPClient.h>

// Wi-Fi Credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Server Endpoint (Replace with your backend/Firebase/ThingSpeak URL)
const char* serverUrl = "http://your-server-ip:5000/api/flood-data";

// Pin Definitions
#define TRIG_PIN 5
#define ECHO_PIN 18
#define BUZZER_PIN 19
#define RED_LED 21
#define GREEN_LED 22

// Tank / Riverbed Calibration (in cm)
const float SENSOR_HEIGHT_CM = 200.0; // Distance from sensor to empty riverbed
const float WARNING_THRESHOLD = 120.0; // Water level for Warning (cm)
const float DANGER_THRESHOLD = 160.0;  // Water level for Critical Alert (cm)

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected! IP Address: " + WiFi.localIP().toString());
}

float getWaterLevel() {
  // Clear trigger
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  
  // Send 10us pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  // Read echo duration
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout
  float distance = duration * 0.034 / 2.0; // Convert to cm
  
  // Calculate actual water height
  float waterLevel = SENSOR_HEIGHT_CM - distance;
  return (waterLevel < 0) ? 0 : waterLevel;
}

void triggerLocalAlarms(String status) {
  if (status == "CRITICAL") {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(GREEN_LED, LOW);
    // Rapid alarm beep
    tone(BUZZER_PIN, 2000); 
  } else if (status == "WARNING") {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(GREEN_LED, HIGH);
    // Intermittent beep
    tone(BUZZER_PIN, 1000, 300);
  } else {
    digitalWrite(RED_LED, LOW);
    digitalWrite(GREEN_LED, HIGH);
    noTone(BUZZER_PIN);
  }
}

void sendDataToCloud(float level, String status) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(serverUrl);
    http.addHeader("Content-Type", "application/json");

    String jsonPayload = "{\"sensor_id\":\"NODE_01\",\"water_level\":" + String(level) + ",\"status\":\"" + status + "\"}";
    int httpResponseCode = http.POST(jsonPayload);

    if (httpResponseCode > 0) {
      Serial.println("Data sent successfully. Response: " + String(httpResponseCode));
    } else {
      Serial.println("Error sending data: " + String(httpResponseCode));
    }
    http.end();
  }
}

void loop() {
  float currentLevel = getWaterLevel();
  String alertStatus = "SAFE";

  if (currentLevel >= DANGER_THRESHOLD) {
    alertStatus = "CRITICAL";
  } else if (currentLevel >= WARNING_THRESHOLD) {
    alertStatus = "WARNING";
  }

  Serial.printf("Water Level: %.2f cm | Status: %s\n", currentLevel, alertStatus.c_str());

  triggerLocalAlarms(alertStatus);
  sendDataToCloud(currentLevel, alertStatus);

  delay(3000); // Read every 3 seconds
}


/*
 * SCD40 CO2/Temperature/Humidity Sensor - Web Server for ESP32
 * * Required Libraries:
 * "Sensirion I2C SCD4x" by Sensirion
 * WiFi (Built-in for ESP32)
 * WebServer (Built-in for ESP32)
 */

#include <Arduino.h>
#include <Wire.h>
#include <SensirionI2cScd4x.h>
#include <WiFi.h>
#include <WebServer.h>

// --- WiFi Credentials ---
const char* ssid = "WLAN-937955";
const char* password = "21809460619420112131";

// --- I2C Pins ---
#define SDA_PIN 21
#define SCL_PIN 22

// --- LED Pins ---
#define GREEN_LED_PIN 32
#define RED_LED_PIN 33

SensirionI2cScd4x scd4x;
WebServer server(80); // Start server on port 80 (Standard HTTP)

// --- Global variables to store the latest readings ---
uint16_t current_co2 = 0;
float current_temperature = 0.0f;
float current_humidity = 0.0f;

// --- Timing variables for non-blocking delays ---
unsigned long lastMeasurementTime = 0;
const unsigned long measurementInterval = 5000; // 5 seconds

unsigned long lastBlinkTime = 0;
const unsigned long blinkInterval = 500; // 500 milliseconds (half a second)
bool redLedState = false; // Tracks whether the red LED is currently on or off

static char errorMessage[64];
static int16_t error;

void printError(const char* context, int16_t err) {
  Serial.print(context);
  errorToString(err, errorMessage, sizeof(errorMessage));
  Serial.println(errorMessage);
}

// --- HTML Dashboard Generation ---
void handleRoot() {
  // Build the HTML page. It includes CSS for styling and a meta tag to auto-refresh every 5 seconds.
  String html = "<!DOCTYPE html><html><head>";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">";
  html += "<meta http-equiv=\"refresh\" content=\"5\">"; // Auto-refresh page every 5 seconds
  html += "<title>ESP32 Air Quality</title>";
  html += "<style>";
  html += "body { font-family: 'Segoe UI', Arial, sans-serif; text-align: center; background-color: #f4f4f9; color: #333; margin-top: 50px; }";
  html += "h1 { color: #2c3e50; }";
  html += ".container { display: flex; flex-direction: column; align-items: center; gap: 20px; }";
  html += ".card { background: white; padding: 20px; width: 300px; border-radius: 12px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }";
  html += ".card h2 { margin: 0; font-size: 1.2rem; color: #7f8c8d; }";
  html += ".value { font-size: 2.5rem; font-weight: bold; margin: 10px 0 0 0; color: #2980b9; }";
  html += ".co2 { color: #e74c3c; }";
  html += ".hum { color: #27ae60; }";
  html += "</style></head><body>";
  
  html += "<h1>ESP32 Air Quality Monitor</h1>";
  html += "<div class=\"container\">";
  
  // CO2 Card
  html += "<div class=\"card\"><h2>CO2 Level</h2><p class=\"value co2\">";
  html += current_co2 > 0 ? String(current_co2) + " ppm" : "--";
  html += "</p></div>";

  // Temperature Card
  html += "<div class=\"card\"><h2>Temperature</h2><p class=\"value\">";
  html += current_temperature != 0.0 ? String(current_temperature, 1) + " &deg;C" : "--";
  html += "</p></div>";

  // Humidity Card
  html += "<div class=\"card\"><h2>Humidity</h2><p class=\"value hum\">";
  html += current_humidity != 0.0 ? String(current_humidity, 1) + " %" : "--";
  html += "</p></div>";

  html += "</div></body></html>";

  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  while (!Serial) { delay(100); }
  delay(1000);

  Serial.println("Starting SCD40 & Web Server...");

  // --- Initialize LEDs ---
  pinMode(GREEN_LED_PIN, OUTPUT);
  pinMode(RED_LED_PIN, OUTPUT);
  digitalWrite(GREEN_LED_PIN, LOW);
  digitalWrite(RED_LED_PIN, LOW);

  // --- Initialize WiFi ---
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("Connected! IP Address: ");
  Serial.println(WiFi.localIP());

  // --- Initialize I2C and Sensor ---
  Wire.begin(SDA_PIN, SCL_PIN);
  scd4x.begin(Wire, SCD41_I2C_ADDR_62);

  error = scd4x.stopPeriodicMeasurement();
  if (error != 0) { printError("Error stopping previous measurement: ", error); }
  delay(500);

  error = scd4x.startPeriodicMeasurement();
  if (error != 0) {
    printError("Error starting periodic measurement: ", error);
  } else {
    Serial.println("Periodic measurement started.");
  }

  // --- Configure and Start Web Server ---
  server.on("/", handleRoot); // When a user visits the root IP, trigger handleRoot()
  server.begin();
  Serial.println("HTTP server started.");
}

void loop() {
  // 1. Constantly handle incoming web requests (Non-blocking)
  server.handleClient();

  // 2. LED Danger Logic (Non-blocking timer)
  if (current_co2 >= 2000) {
    // Danger: Turn off green, blink red
    digitalWrite(GREEN_LED_PIN, LOW);
    
    if (millis() - lastBlinkTime >= blinkInterval) {
      lastBlinkTime = millis();
      redLedState = !redLedState; // Flip the state
      digitalWrite(RED_LED_PIN, redLedState ? HIGH : LOW);
    }
  } else if (current_co2 > 0) {
    // Safe: Solid green, red off
    digitalWrite(RED_LED_PIN, LOW);
    digitalWrite(GREEN_LED_PIN, HIGH);
  }

  // 3. Check the sensor only if 5 seconds have passed (Non-blocking timer)
  if (millis() - lastMeasurementTime >= measurementInterval) {
    lastMeasurementTime = millis(); // Reset the timer

    bool isDataReady = false;
    error = scd4x.getDataReadyStatus(isDataReady);
    if (error == 0 && isDataReady) {
      uint16_t co2 = 0;
      float temperature = 0.0f;
      float humidity = 0.0f;

      error = scd4x.readMeasurement(co2, temperature, humidity);
      if (error == 0 && co2 != 0) {
        // Update the global variables so the web server can serve the latest data
        current_co2 = co2;
        current_temperature = temperature;
        current_humidity = humidity;

        // Still print to serial for debugging
        Serial.print("CO2: "); Serial.print(co2);
        Serial.print("\tTemp: "); Serial.print(temperature, 1);
        Serial.print("\tHum: "); Serial.println(humidity, 1);
      }
    }
  }
}

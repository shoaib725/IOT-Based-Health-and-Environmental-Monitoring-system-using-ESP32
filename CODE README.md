/*
 * Health Monitor v2.8 (Enhanced LED Control & Fixed Blynk Mapping)
 * Features:
 * - Works WITHOUT WiFi/Blynk (Offline Mode)
 * - Full OLED display shows all sensor data
 * - Dual Alert: Gas High OR Touch Button
 * - Auto-connects to Blynk when WiFi available
 * - Modified: Page 0 (HR/SpO2) displays for 30 seconds
 * - ENHANCED: Medical-grade ECG at 250Hz with bandpass filtering
 * - NEW: Smart LED control with Blynk override
 * - FIXED: Correct Blynk datastream mapping (V2, V13, V14)
 * - REMOVED: MPU6050 accelerometer (not needed)
 */

// ============================================================================
// BLYNK CONFIGURATION
// ============================================================================
#define BLYNK_TEMPLATE_ID "TMPL34tvcgOtB"
#define BLYNK_TEMPLATE_NAME "Health Monitor"
#define BLYNK_AUTH_TOKEN "LIaq8iJ-nZRwnRHUAiyD-sUd_121KWGP"
#define BLYNK_PRINT Serial

#include <Wire.h>
#include <WiFi.h>
#include <BlynkSimpleEsp32.h> 
#include <TinyGPSPlus.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "MAX30105.h"
#include "heartRate.h"
#include "spo2_algorithm.h"
#include "DHT.h"
#include <OneWire.h>
#include <DallasTemperature.h>

// ---------------- CONNECTIVITY STATUS ----------------
bool wifiConnected = false;
bool blynkConnected = false;
unsigned long lastReconnectAttempt = 0;
const unsigned long RECONNECT_INTERVAL = 30000; // Try reconnect every 30s

// ---------------- LED CONTROL MODE ----------------
bool blynkLedOverride = false;  // true = Blynk button pressed (force LED ON)

// ---------------- ALERTS CONFIGURATION ----------------
unsigned long lastAlertTime = 0;
const unsigned long ALERT_COOLDOWN = 30000; // 30 seconds cooldown between alerts
const int GAS_MAX_LIMIT = 200; 

// ---------------- PINS & SENSORS ----------------
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define BUZZER_PIN 26
#define LED_PIN 32
#define LDR_PIN 33
#define ece 15
#define TOUCH_PIN 4 
#define GPS_RX 16
#define GPS_TX 17
#define DHTPIN 27
#define DHTTYPE DHT11
#define GAS_PIN 35
#define ECG_PIN 34
#define DS18B20_PIN 25

// ---------------- ECG CONFIGURATION (ENHANCED) ----------------
#define FILTER_SIZE 10              // Increased from 5 for better filtering
#define ECG_SAMPLE_RATE 4           // 4ms = 250Hz (medical grade)
#define BLYNK_ECG_RATE 100          // Send to Blynk every 100ms (rate limiting)

int ecgBuffer[FILTER_SIZE] = {0};
int bufferIndex = 0;
int ecgBaseline = 2048;             // Dynamic baseline tracker
unsigned long lastECGSample = 0;
unsigned long lastBlynkECG = 0;

// Bandpass filter states for P and T wave visibility
float filterHP = 0.0;               // High-pass filter state
float filterLP = 0.0;               // Low-pass filter state
const float HP_ALPHA = 0.996;       // High-pass: removes drift below 0.5Hz
const float LP_ALPHA = 0.1;         // Low-pass: removes noise above 40Hz

// ---------------- OBJECTS ----------------
HardwareSerial gpsSerial(2);
TinyGPSPlus gps;
DHT dht(DHTPIN, DHTTYPE);
OneWire oneWire(DS18B20_PIN);
DallasTemperature ds18b20(&oneWire);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
MAX30105 particleSensor;
BlynkTimer timer;

// ---------------- VARIABLES ----------------
float ds18b20Temp = 0;
const char* ssid = "Realme 6";
const char* password = "1234567890@";

#define BUFFER_SIZE 100
uint32_t irBuffer[BUFFER_SIZE];
uint32_t redBuffer[BUFFER_SIZE];
int32_t spo2;
int8_t validSPO2;
int32_t heartRate;
int8_t validHeartRate;

// Display page tracking - MODIFIED FOR CUSTOM TIMING
int displayPage = 0;
unsigned long lastPageSwitch = 0;
const unsigned long PAGE0_INTERVAL = 30000; // 30 seconds for vitals page
const unsigned long PAGE_SWITCH_INTERVAL = 3000; // 3 seconds for other pages

// Sensor data storage
int displayHR = 0;
int displaySpO2 = 0;
float maxTemp = 0;
float dhtTemp = 0;
float humidity = 0;
int gasValue = 0;
int ldrValue = 0;
int touchValue = 0;
float latitude = 0.0;
float longitude = 0.0;

// ============================================================================
// ENHANCED ECG FILTERING FUNCTION
// ============================================================================
int readECGFiltered() {
  // Read raw ADC value
  int raw = analogRead(ECG_PIN);
  
  // 1. Moving average filter (10-sample smoothing)
  ecgBuffer[bufferIndex] = raw;
  bufferIndex = (bufferIndex + 1) % FILTER_SIZE;
  
  int sum = 0;
  for (int i = 0; i < FILTER_SIZE; i++) {
    sum += ecgBuffer[i];
  }
  float filtered = sum / (float)FILTER_SIZE;
  
  // 2. High-pass filter (removes DC drift and breathing artifacts)
  //    This makes P and T waves visible!
  filterHP = HP_ALPHA * filterHP + HP_ALPHA * (filtered - ecgBaseline);
  ecgBaseline = filtered;
  
  // 3. Low-pass filter (removes muscle noise and 50/60Hz interference)
  filterLP = LP_ALPHA * filterHP + (1.0 - LP_ALPHA) * filterLP;
  
  // 4. Scale and center for display
  int output = (int)(filterLP) + 2048;
  
  // 5. Clamp to valid ADC range
  if (output < 0) output = 0;
  if (output > 4095) output = 4095;
  
  return output;
}

// ============================================================================
// SMART LED CONTROL WITH BLYNK OVERRIDE
// ============================================================================
void updateLED() {
  if (blynkLedOverride) {
    // Blynk button pressed - Force LED ON
    digitalWrite(LED_PIN, HIGH);
  } else {
    // Auto mode - LED follows LDR (dark room = LED ON)
    digitalWrite(LED_PIN, ldrValue > 400 ? HIGH : LOW);
  }
}

// WiFi Connection (Non-blocking)
bool connectWiFi() {
  Serial.println("Connecting to WiFi...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  unsigned long startTime = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - startTime < 10000) {
    delay(500);
    Serial.print(".");
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected!");
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
    wifiConnected = true;
    return true;
  } else {
    Serial.println("\nWiFi Failed - Running Offline");
    wifiConnected = false;
    return false;
  }
}

// Blynk Connection (Non-blocking)
bool connectBlynk() {
  if (!wifiConnected) return false;
  
  Serial.println("Connecting to Blynk...");
  Blynk.config(BLYNK_AUTH_TOKEN);
  
  unsigned long startTime = millis();
  while (!Blynk.connect() && millis() - startTime < 5000) {
    delay(500);
    Serial.print(".");
  }
  
  if (Blynk.connected()) {
    Serial.println("\nBlynk Connected!");
    blynkConnected = true;
    return true;
  } else {
    Serial.println("\nBlynk Failed - Data stored locally");
    blynkConnected = false;
    return false;
  }
}

// Check and reconnect if needed
void checkConnectivity() {
  if (millis() - lastReconnectAttempt < RECONNECT_INTERVAL) return;
  
  lastReconnectAttempt = millis();
  
  if (WiFi.status() != WL_CONNECTED) {
    wifiConnected = false;
    blynkConnected = false;
    Serial.println("WiFi lost, attempting reconnect...");
    connectWiFi();
  }
  
  if (wifiConnected && !Blynk.connected()) {
    blynkConnected = false;
    Serial.println("Blynk lost, attempting reconnect...");
    connectBlynk();
  }
}

// *** DUAL TRIGGER ALERT LOGIC (Works Offline & Online) ***
void checkSafetyTriggers() {
  // 1. Check Cooldown (Prevents spamming)
  if (millis() - lastAlertTime < ALERT_COOLDOWN) return;

  String alertMsg = "";
  bool trigger = false;
  int beepCount = 0;

  // 2. CONDITION A: Is Gas High? (Automatic Danger)
  if (gasValue > GAS_MAX_LIMIT) {
    alertMsg = "CRITICAL: High Gas Level (" + String(gasValue) + ")";
    trigger = true;
    beepCount = 5; // Long urgent beep sequence
  }
  
  // 3. CONDITION B: Is Touch Pressed? (Manual Help)
  else if (touchValue == HIGH) {
    alertMsg = "ALERT: Emergency Button Pressed!";
    trigger = true;
    beepCount = 3; // Short manual beep sequence
  }

  // 4. Send Alert if EITHER condition happened
  if (trigger) {
    Serial.println(">>> ALERT: " + alertMsg);
    
    // Send to Blynk if connected
    if (blynkConnected) {
      Blynk.logEvent("vital_alert", alertMsg); 
      Serial.println("    -> Sent to Blynk");
    } else {
      Serial.println("    -> Offline: Alert logged locally");
    }
    
    // Buzz Alarm (Works offline)
    for(int i=0; i<beepCount; i++) {
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    }
    
    lastAlertTime = millis(); // Reset timer
  }
}

// Multi-page OLED Display - MODIFIED FOR CUSTOM PAGE TIMING
void updateOLED() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  
  // Status Bar (Always visible)
  display.print("W:");
  display.print(wifiConnected ? "OK" : "X");
  display.print(" B:");
  display.print(blynkConnected ? "OK" : "X");
  display.print(" P:");
  display.println(displayPage + 1);
  display.println("-------------------");
  
  // Auto-switch pages with custom timing
  unsigned long currentInterval;
  if (displayPage == 0) {
    currentInterval = PAGE0_INTERVAL; // 30 seconds for vitals page
  } else {
    currentInterval = PAGE_SWITCH_INTERVAL; // 3 seconds for other pages
  }
  
  if (millis() - lastPageSwitch > currentInterval) {
    displayPage = (displayPage + 1) % 3; // 3 pages total
    lastPageSwitch = millis();
  }
  
  // PAGE 0: Vitals (30 second display)
  if (displayPage == 0) {
    long irValue = particleSensor.getIR();
    if (irValue < 50000) {
      display.println("Place Finger...");
    } else {
      display.print("HR: ");
      display.print(displayHR);
      display.println(" bpm");
      
      display.print("SpO2: ");
      display.print(displaySpO2);
      display.println(" %");
    }
    
    display.print("Body T: ");
    if (ds18b20Temp > 0) {
      display.print(ds18b20Temp, 1);
      display.println(" C");
    } else {
      display.println("--");
    }
    
    display.print("DHT T: ");
    display.print(dhtTemp, 1);
    display.println(" C");
    
    display.print("Humidity: ");
    display.print(humidity, 0);
    display.println(" %");
  }
  
  // PAGE 1: Environment & Safety (3 second display)
  else if (displayPage == 1) {
    // Alert Status
    if (gasValue > GAS_MAX_LIMIT) {
      display.println("!! GAS DANGER !!");
      display.setTextSize(2);
      display.print(gasValue);
      display.setTextSize(1);
      display.println(" ppm");
    } else if (touchValue == HIGH) {
      display.println("!! BUTTON ALERT !!");
    } else {
      display.println("System Safe");
    }
    display.println("");
    
    display.print("Gas: ");
    display.print(gasValue);
    display.println(" ppm");
    
    display.print("Light: ");
    display.println(ldrValue);
    
    display.print("LED: ");
    display.print(digitalRead(LED_PIN) ? "ON" : "OFF");
    display.print(" (");
    display.print(blynkLedOverride ? "BLYNK" : "AUTO");
    display.println(")");
    
    display.print("Touch: ");
    display.println(touchValue ? "YES" : "NO");
  }
  
  // PAGE 2: GPS & System (3 second display)
  else if (displayPage == 2) {
    display.println("GPS Location:");
    
    if (gps.location.isValid()) {
      display.print("Lat: ");
      display.println(latitude, 6);
      
      display.print("Lng: ");
      display.println(longitude, 6);
      
      display.print("Sats: ");
      display.println(gps.satellites.value());
    } else {
      display.println("No GPS Fix");
      display.println("Searching...");
    }
    display.println("");
    
    display.print("Uptime: ");
    display.print(millis() / 60000);
    display.println(" min");
    
    if (wifiConnected) {
      display.print("RSSI: ");
      display.print(WiFi.RSSI());
      display.println(" dBm");
    }
  }
  
  display.display();
}

// Send data to Blynk (only if connected) - FIXED MAPPING
void sendToBlynk() {
  if (!blynkConnected) return;
  
  // Health Vitals
  Blynk.virtualWrite(V0, displayHR);           // Heart Rate
  Blynk.virtualWrite(V1, displaySpO2);         // SpO2
  Blynk.virtualWrite(V2, ds18b20Temp);         // Body Temp (DS18B20)
  
  // Environment
  Blynk.virtualWrite(V3, dhtTemp);             // Ambient Temp (DHT11)
  Blynk.virtualWrite(V4, humidity);            // Humidity
  Blynk.virtualWrite(V5, gasValue);            // Gas Level
  Blynk.virtualWrite(V6, ldrValue);            // Light Level
  
  // Sensors
  Blynk.virtualWrite(V8, touchValue);          // Touch Sensor
  Blynk.virtualWrite(V9, latitude);            // Latitude
  Blynk.virtualWrite(V15, longitude);          // Longitude
  
  // System Info
  Blynk.virtualWrite(V13, millis() / 1000);    // System Uptime (seconds)
  Blynk.virtualWrite(V14, WiFi.RSSI());        // WiFi Signal Strength
  
  // Sync LED state to Blynk
  Blynk.virtualWrite(V10, digitalRead(LED_PIN));
}

// ---------------- SETUP ----------------
void setup() {
  Serial.begin(115200);
  Serial.println("\n=== Health Monitor v2.8 ===");
  Serial.println("Offline-First Design");
  Serial.println("Page 0: 30s display");
  Serial.println("ECG: 250Hz + Bandpass Filter");
  Serial.println("LED: Smart Override Control");
  Serial.println("Blynk: Fixed Data Mapping");
  Serial.println("MPU6050: REMOVED");
  
  analogReadResolution(12);
  Wire.begin(21, 22);

  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  pinMode(GAS_PIN, INPUT);
  pinMode(LDR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(ece, INPUT_PULLUP);
  pinMode(TOUCH_PIN, INPUT);

  gpsSerial.begin(9600, SERIAL_8N1, GPS_RX, GPS_TX);

  // OLED Init
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED init failed!");
  } else {
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.clearDisplay();
    display.println("Health Monitor v2.8");
    display.println("Enhanced LED Control");
    display.println("No Accelerometer");
    display.println("Initializing...");
    display.display();
  }

  // Sensors Init
  particleSensor.begin(Wire, I2C_SPEED_STANDARD);
  particleSensor.setup();
  particleSensor.setPulseAmplitudeRed(0x1F);
  particleSensor.setPulseAmplitudeIR(0x1F);
  particleSensor.setPulseAmplitudeGreen(0);

  dht.begin();
  ds18b20.begin();

  // Try WiFi/Blynk (Non-blocking)
  display.clearDisplay();
  display.setCursor(0,0);
  display.println("Connecting WiFi...");
  display.display();
  
  if (connectWiFi()) {
    display.println("WiFi OK");
    display.println("Connecting Blynk...");
    display.display();
    connectBlynk();
  }

  display.clearDisplay();
  display.setCursor(0,0);
  display.println("System Ready!");
  display.print("WiFi: ");
  display.println(wifiConnected ? "YES" : "NO");
  display.print("Blynk: ");
  display.println(blynkConnected ? "YES" : "NO");
  display.println("");
  display.println("Running in");
  display.println(wifiConnected ? "ONLINE Mode" : "OFFLINE Mode");
  display.display();
  delay(3000);
  
  Serial.println("=== Setup Complete ===");
  Serial.print("WiFi: ");
  Serial.println(wifiConnected ? "Connected" : "Offline");
  Serial.print("Blynk: ");
  Serial.println(blynkConnected ? "Connected" : "Offline");
}

// ---------------- LOOP ----------------
void loop() {
  // Run Blynk only if connected
  if (blynkConnected) Blynk.run(); 
  
  // Check connectivity periodically
  checkConnectivity();
  
  // GPS Update
  while (gpsSerial.available()) gps.encode(gpsSerial.read());
  latitude = gps.location.isValid() ? gps.location.lat() : 0.0;
  longitude = gps.location.isValid() ? gps.location.lng() : 0.0;

  // MAX30102 Heart Rate & SpO2
  displayHR = 0;
  displaySpO2 = 0;
  long irValue = particleSensor.getIR();
  
  if (irValue > 50000) {
    for (byte i = 0; i < BUFFER_SIZE; i++) {
      while (!particleSensor.available()) particleSensor.check();
      redBuffer[i] = particleSensor.getRed();
      irBuffer[i] = particleSensor.getIR();
      particleSensor.nextSample();
    }
    maxim_heart_rate_and_oxygen_saturation(
      irBuffer, BUFFER_SIZE, redBuffer, 
      &spo2, &validSPO2, 
      &heartRate, &validHeartRate
    );
    if (validHeartRate) displayHR = heartRate;
    if (validSPO2) displaySpO2 = spo2;
  }

  // Temperature Sensors
  maxTemp = particleSensor.readTemperature();
  dhtTemp = dht.readTemperature();
  humidity = dht.readHumidity();
  if (isnan(dhtTemp) || isnan(humidity)) { 
    dhtTemp = 0; 
    humidity = 0; 
  }

  ds18b20.requestTemperatures();
  ds18b20Temp = ds18b20.getTempCByIndex(0);
  if (ds18b20Temp == DEVICE_DISCONNECTED_C) ds18b20Temp = 0;

  // Environmental Sensors
  gasValue = analogRead(GAS_PIN) / 10.23;
  ldrValue = analogRead(LDR_PIN);
  
  // *** SMART LED CONTROL - USES NEW FUNCTION ***
  updateLED();

  // ============================================================================
  // ENHANCED ECG RECORDING (250Hz + Bandpass Filter)
  // ============================================================================
  if (digitalRead(ece) == LOW) {
    Serial.println("ECG Recording... (250Hz, Medical-Grade)");
    
    while (digitalRead(ece) == LOW) {
      // High-speed sampling at 250Hz
      if (millis() - lastECGSample >= ECG_SAMPLE_RATE) {
        lastECGSample = millis();
        
        int ecg_val = readECGFiltered();
        
        // Print to Serial Plotter (shows P-QRS-T waves)
        Serial.println(ecg_val);
        
        // Rate-limited Blynk updates (10Hz max)
        if (blynkConnected && (millis() - lastBlynkECG >= BLYNK_ECG_RATE)) {
          lastBlynkECG = millis();
          Blynk.virtualWrite(V12, ecg_val);
        }
        
        // Keep Blynk connection alive
        if (blynkConnected) Blynk.run();
      }
    }
    
    Serial.println("ECG Stopped");
  }

  // Touch Sensor
  touchValue = digitalRead(TOUCH_PIN);

  // Update OLED Display (Works offline)
  updateOLED();

  // Check Safety Triggers (Works offline)
  checkSafetyTriggers();

  // Send to Blynk (Only if connected)
  sendToBlynk();

  delay(1500); 
}

// ============================================================================
// BLYNK LED CONTROL (V10) - ENHANCED WITH OVERRIDE
// ============================================================================
BLYNK_WRITE(V10) { 
  int buttonState = param.asInt();
  
  if (buttonState == 1) {
    // Button pressed - Override to ON
    blynkLedOverride = true;
    digitalWrite(LED_PIN, HIGH);
    Serial.println("LED: Blynk Override ON (forced)");
  } else {
    // Button released - Return to AUTO mode
    blynkLedOverride = false;
    Serial.println("LED: Returned to AUTO mode (LDR control)");
  }
}

BLYNK_WRITE(V11) { 
  if (param.asInt()) { 
    digitalWrite(BUZZER_PIN, HIGH); 
    delay(100); 
    digitalWrite(BUZZER_PIN, LOW); 
    Serial.println("Buzzer test via Blynk");
  }
}

// ============================================================================
// VERSION 2.8 CHANGELOG:
// ============================================================================
// ✅ FIXED: LED control now persistent (no race condition)
// ✅ FIXED: V2 now sends DS18B20 body temp (was MAX30102 temp)
// ✅ FIXED: V13 now sends system uptime in seconds (was body temp)
// ✅ ADDED: V14 now sends WiFi RSSI signal strength
// ✅ REMOVED: V7 acceleration (not in your Blynk datastreams)
// ✅ REMOVED: MPU6050 library and all accelerometer code
// ✅ ENHANCED: OLED shows LED control mode (AUTO/BLYNK)
// ✅ ENHANCED: updateLED() function for clean control logic
// 
// REMOVED COMPONENTS:
// - #include <MPU6050.h>
// - MPU6050 mpu object
// - mpu.initialize() in setup
// - accX variable and reading
// - AccX display on OLED PAGE 1
// 
// BEHAVIOR:
// - Default: LED follows LDR (dark=ON, bright=OFF)
// - Blynk ON: LED forced ON (overrides LDR)
// - Blynk OFF: LED returns to AUTO mode
// 
// SENSORS ACTIVE:
// ✅ MAX30105 (Heart Rate, SpO2, Temperature)
// ✅ DS18B20 (Body Temperature)
// ✅ DHT11 (Room Temperature, Humidity)
// ✅ MQ Gas Sensor
// ✅ LDR Light Sensor
// ✅ Touch Sensor
// ✅ GPS Module
// ✅ ECG Sensor (250Hz filtered)
// ❌ MPU6050 Accelerometer (REMOVED)
// ============================================================================

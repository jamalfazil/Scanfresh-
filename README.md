# Scanfresh-
I am doing a project in finding expiry product I have a problem in connecting with my server can you solve that I just want a solution I don't know how to do can you give me some suggestion is I will upload my code below
#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <RTClib.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>
#include <BLEClient.h>

// ----- I2C Pin Definitions for RTC (DS3231) on ESP32-S3 DevKitC-1 -----
#define I2C_SDA 8   // Adjust these according to your board's pinout
#define I2C_SCL 9

RTC_DS3231 rtc;

// Wi-Fi and Notification Server settings
const char* ssid = "YourWiFiSSID";
const char* password = "YourWiFiPassword";
const char* server = "https://your-notification-server.com/send";

// BLE Service & Characteristic UUIDs (for communication with Fridge Module)
#define SERVICE_UUID        "12345678-1234-5678-1234-56789abcdef0"
#define CHARACTERISTIC_UUID "abcdef01-1234-5678-1234-56789abcdef0"

BLEClient* bleClient = nullptr;
BLERemoteCharacteristic* remoteCharacteristic = nullptr;

// ----- Product Data Structure -----
struct Product {
  int id;
  int year;
  int month;
  int day;
  String name;   // Optional: product name
};

// Example: One product ("Milk") with an expiry date.
// (Set an expiry date for testing; adjust year/month/day as needed)
Product products[10] = {
  {1, 2025, 5, 1, "Milk"}
};
int productCount = 1;

// ----- Function Prototypes -----
void sendNotification(String message);
void checkExpiry();
void connectToFridge();
void receiveTemperature();

void setup() {
  Serial.begin(115200);

  // Initialize I2C with defined pins for RTC
  Wire.begin(I2C_SDA, I2C_SCL);

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi Connected");

  // Initialize RTC
  if (!rtc.begin()) {
    Serial.println("Couldn't find RTC");
    while (1);
  }
  if (rtc.lostPower()) {
    Serial.println("RTC lost power, setting time to compile time...");
    rtc.adjust(DateTime(__DATE__, __TIME__));
  }

  // Initialize BLE as a client to connect to the Fridge Module
  BLEDevice::init("ESP32S3_Main");
  bleClient = BLEDevice::createClient();

  // Optionally, connect to the fridge module at startup
  // (You can also call connectToFridge() later when needed)
  connectToFridge();
}

void loop() {
  // Check for product expiry notifications based on RTC
  checkExpiry();

  // Read temperature from Fridge Module over BLE (if connected)
  receiveTemperature();

  // Optionally, you could attempt to reconnect to the fridge module if needed:
  if (remoteCharacteristic == nullptr) {
    connectToFridge();
  }
  
  // Run every 1 minute
  delay(60000);
}

void checkExpiry() {
  DateTime now = rtc.now();

  for (int i = 0; i < productCount; i++) {
    DateTime expiry(products[i].year, products[i].month, products[i].day);
    TimeSpan diff = expiry - now;
    // Send a notification 7 days before expiry
    if (diff.days() == 7) {
      sendNotification("Reminder: " + products[i].name + " (ID " + String(products[i].id) + ") will expire in 7 days.");
    }
    // Send a notification if the product is expired or on the day of expiry
    else if (diff.days() <= 0) {
      sendNotification("Alert: " + products[i].name + " (ID " + String(products[i].id) + ") has expired!");
    }
  }
}

void connectToFridge() {
  BLEScan* pBLEScan = BLEDevice::getScan();
  pBLEScan->setActiveScan(true);
  BLEScanResults* foundDevices = pBLEScan->start(5, false); // 5-second scan, passive mode

  if (foundDevices) {  // Check if we got scan results
    for (int i = 0; i < foundDevices->getCount(); i++) {
      BLEAdvertisedDevice d = foundDevices->getDevice(i);
      if (d.haveServiceUUID() && d.isAdvertisingService(BLEUUID(SERVICE_UUID))) {
        Serial.println("Found Fridge Module, connecting...");
        if (bleClient->connect(&d)) {
          BLERemoteService* remoteService = bleClient->getService(BLEUUID(SERVICE_UUID));
          if (remoteService) {
            remoteCharacteristic = remoteService->getCharacteristic(BLEUUID(CHARACTERISTIC_UUID));
            if (remoteCharacteristic) {
              Serial.println("Connected to Fridge Module!");
            }
          }
        }
        break;  // Stop scanning after connecting
      }
    }
  } else {
    Serial.println("No BLE devices found.");
  }
}

void receiveTemperature() {
  // If connected via BLE and the characteristic is readable, get the temperature value.
  if (remoteCharacteristic && remoteCharacteristic->canRead()) {
    String tempStr = remoteCharacteristic->readValue();
    float temperature = tempStr.toFloat();
    
    if (temperature > 10.0) {  // Example threshold
      sendNotification("Alert: Fridge temperature too high: " + String(temperature) + "°C");
    }
  }
}

void sendNotification(String message) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(server);
    http.addHeader("Content-Type", "application/json");
    String payload = "{\"message\": \"" + message + "\"}";
    http.POST(payload);
    http.end();
    Serial.println("Notification sent: " + message);
  } else {
    Serial.println("Wi-Fi not connected; notification not sent.");
  }
}

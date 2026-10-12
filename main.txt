/*
  Raspberry Pi Pico + PN532 NFC Logger
  Logs scanned NFC tags into a 'logs.json' file using LittleFS and ArduinoJson.
*/

#include <Wire.h>
#include <Adafruit_PN532.h>
#include <LittleFS.h>
#include <ArduinoJson.h>

#define I2C_SDA 4
#define I2C_SCL 5
Adafruit_PN532 nfc(I2C_SDA, I2C_SCL);

const char* filename = "/logs.json";

void setup(void) {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) delay(10);

  // Initialize LittleFS
  if (!LittleFS.begin()) {
    Serial.println("An Error has occurred while mounting LittleFS. Formatting...");
    LittleFS.format();
    LittleFS.begin();
  }
  Serial.println("LittleFS Mounted Successfully.");

  // Initialize PN532
  nfc.begin();
  uint32_t versiondata = nfc.getFirmwareVersion();
  if (!versiondata) {
    Serial.print("Didn't find PN532 board. Check wiring!");
    while (1);
  }
  nfc.setPassiveActivationRetries(0xFF);
  nfc.SAMConfig();
  
  Serial.println("Ready! Waiting for an NFC Card/Tag to log...");
}

void logTagToJson(String uidStr) {
  // 1. Read existing JSON file or create a new array if empty
  File file = LittleFS.open(filename, "r");
  JsonDocument doc;
  
  if (file) {
    DeserializationError error = deserializeJson(doc, file);
    file.close();
    if (error) {
      Serial.println("Failed to read file, creating new log list.");
      doc.to<JsonArray>();
    }
  } else {
    doc.to<JsonArray>(); // Initialize as array if file doesn't exist
  }

  JsonArray array = doc.as<JsonArray>();

  // 2. Create a new log entry object
  JsonObject entry = array.add<JsonObject>();
  entry["timestamp_ms"] = millis();
  entry["uid"] = uidStr;

  // 3. Save the updated JSON back to flash memory
  file = LittleFS.open(filename, "w");
  if (!file) {
    Serial.println("Failed to open file for writing!");
    return;
  }

  if (serializeJson(doc, file) == 0) {
    Serial.println("Failed to write JSON to file.");
  } else {
    Serial.print("Logged successfully: ");
    serializeJson(entry, Serial);
    Serial.println();
  }
  file.close();
}

void loop(void) {
  boolean success;
  uint8_t uid[] = { 0, 0, 0, 0, 0, 0, 0 };  
  uint8_t uidLength;                        
  
  success = nfc.readPassiveTargetID(PN532_MIFARE_ISO14443A, &uid, &uidLength);
  
  if (success) {
    // Convert UID bytes to a clean Hex string (e.g., "01:23:45:67")
    String uidStr = "";
    for (uint8_t i = 0; i < uidLength; i++) {
      if (uid[i] < 0x10) uidStr += "0";
      uidStr += String(uid[i], HEX);
      if (i < uidLength - 1) uidStr += ":";
    }
    uidStr.toUpperCase();

    // Log it to our JSON file
    logTagToJson(uidStr);
    
    delay(2000); // 2-second cooldown to avoid duplicate logs from one tap
  }
}

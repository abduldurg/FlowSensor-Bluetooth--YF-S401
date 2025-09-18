#include "BluetoothSerial.h"
#include <Preferences.h>

#define FLOW_SENSOR_PIN 14  
#define OUTPUT_PIN 25        // ✅ LED or indicator pin

volatile unsigned long pulseCount = 0;   // ISR still uses 32-bit (fast)
const float pulsesPerLiter = 1663;     // ✅ Calibrate carefully

double totalLiters = 0;
uint64_t totalPulseCount = 0;            // ✅ 64-bit counter

BluetoothSerial SerialBT;
Preferences prefs;

const unsigned int SAVE_INTERVAL_LITERS = 5;  // ✅ Save every 5 liters
double lastSavedLiters = 0;

// ✅ ISR for pulse counting
void IRAM_ATTR pulseCounter() {
  pulseCount++;
}

// ✅ Format double into Indian style with commas
String formatIndian(double value) {
  char raw[30];
  snprintf(raw, sizeof(raw), "%.3f", value);  // convert to string with 3 decimals

  String str(raw);
  int dotIndex = str.indexOf('.');
  String intPart = (dotIndex == -1) ? str : str.substring(0, dotIndex);
  String fracPart = (dotIndex == -1) ? "" : str.substring(dotIndex);  // includes '.'

  // Insert commas in Indian numbering system
  String result = "";
  int len = intPart.length();

  if (len > 3) {
    int firstGroup = len % 2 == 0 ? 1 : 2;
    result = intPart.substring(0, firstGroup);
    int idx = firstGroup;
    while (idx < len) {
      result += "," + intPart.substring(idx, idx + 2);
      idx += 2;
    }
  } else {
    result = intPart;
  }

  return result + fracPart + " liter";
}

void setup() {
  Serial.begin(115200);
  SerialBT.begin("Abis_FlowSensor");
  Serial.println("Bluetooth started. Pair with Abis_FlowSensor");

  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);
  pinMode(OUTPUT_PIN, OUTPUT);           
  digitalWrite(OUTPUT_PIN, LOW);         

  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), pulseCounter, FALLING);

  // Load saved total from NVS (stored as 64-bit)
  prefs.begin("flowdata", false);
  totalPulseCount = prefs.getULong64("pulses", 0);   // ✅ 64-bit read
  prefs.end();

  totalLiters = (double)totalPulseCount / pulsesPerLiter;
  lastSavedLiters = floor(totalLiters / SAVE_INTERVAL_LITERS) * SAVE_INTERVAL_LITERS;

  Serial.print("Restored Total: ");
  Serial.println(totalLiters, 3);
}

void loop() {
  if (pulseCount > 0) {
    noInterrupts();
    unsigned long currentPulses = pulseCount;
    pulseCount = 0;
    interrupts();

    totalPulseCount += currentPulses;
    totalLiters = (double)totalPulseCount / pulsesPerLiter;

    // ✅ Blink LED for activity
    digitalWrite(OUTPUT_PIN, HIGH);
    delay(20);
    digitalWrite(OUTPUT_PIN, LOW);

    // ✅ Print nicely formatted liters
    String formatted = formatIndian(totalLiters);
    Serial.println(formatted);
    SerialBT.println(formatted);

    // Save only if interval (5L) passed
    if (totalLiters - lastSavedLiters >= SAVE_INTERVAL_LITERS) {
      prefs.begin("flowdata", false);
      prefs.putULong64("pulses", totalPulseCount);   // ✅ 64-bit write
      prefs.end();
      lastSavedLiters = floor(totalLiters / SAVE_INTERVAL_LITERS) * SAVE_INTERVAL_LITERS;

      Serial.println("💾 Progress saved at " + String(lastSavedLiters, 3) + " L");
    }
  }

  // Bluetooth reset command
  if (SerialBT.available()) {
    char cmd = SerialBT.read();
    if (cmd == 'R' || cmd == 'r') {
      totalPulseCount = 0;
      totalLiters = 0.0;
      lastSavedLiters = 0;
      prefs.begin("flowdata", false);
      prefs.putULong64("pulses", 0);   // ✅ reset 64-bit counter
      prefs.end();
      Serial.println("Reset -> 0.000 liter");
      SerialBT.println("Reset -> 0.000 liter");

      digitalWrite(OUTPUT_PIN, LOW);   // turn LED OFF on reset
    }
  }
}

#include <Wire.h>
#include <LiquidCrystal_PCF8574.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>

// ===== LCD Setup =====
LiquidCrystal_PCF8574 lcd(0x27);   // Change to 0x3F if needed

// ===== ADXL345 Setup =====
Adafruit_ADXL345_Unified accel = Adafruit_ADXL345_Unified(12345);

// ===== Sensor Pins =====
#define MOISTURE_PIN A0
#define VIBRATION_PIN A1
#define RAIN_PIN A2
#define ALARM_PIN 7     // Digital output pin for alarm

// ===== Variables =====
int moistureValue = 0;
int vibrationValue = 0;
int rainValue = 0;
float X, Y, Z;

void setup() {
  Serial.begin(9600);
  Wire.begin();

  pinMode(ALARM_PIN, OUTPUT);
  digitalWrite(ALARM_PIN, LOW);  // Alarm OFF at start

  // ===== LCD Initialization =====
  lcd.begin(16, 2);
  lcd.setBacklight(255);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Sensor Monitor");
  delay(1500);
  lcd.clear();

  // ===== ADXL345 Initialization =====
  if (!accel.begin()) {
    Serial.println("No ADXL345 detected!");
    lcd.setCursor(0, 0);
    lcd.print("ADXL345 ERROR!");
    while (1);
  }

  accel.setRange(ADXL345_RANGE_16_G);
  Serial.println("ADXL345 Ready!");
}

void loop() {
  // ===== Read Analog Sensors =====
  int rawMoisture = analogRead(MOISTURE_PIN);
  int rawVibration = analogRead(VIBRATION_PIN);
  int rawRain = analogRead(RAIN_PIN);

  // ===== Scale 0–1023 → 0–100 =====
  moistureValue = map(rawMoisture, 0, 1023, 0, 100);
  moistureValue = 100 - moistureValue; // Invert (dry = low)
  vibrationValue = map(rawVibration, 0, 1023, 0, 100);
  rainValue = map(rawRain, 0, 1023, 0, 100);
  rainValue = 100 - rainValue; // Invert (wet = high)

  // ===== Read ADXL345 =====
  sensors_event_t event;
  accel.getEvent(&event);
  X = event.acceleration.x;
  Y = event.acceleration.y;
  Z = event.acceleration.z;

  // ===== Check Abnormal Conditions =====
  bool alarm = false;

  if (moistureValue > 50) alarm = true;       // Too dry
  if (rainValue > 60) alarm = true;           // Heavy rain
  if (vibrationValue > 70) alarm = true;      // High vibration
  if (abs(X) > 5 || abs(Y) > 5 || abs(Z) > 15) alarm = true; // Strong motion

  // ===== Set Alarm Output =====
  digitalWrite(ALARM_PIN, alarm ? HIGH : LOW);

  // ===== Print to Serial =====
  //Serial.print(",");
  Serial.print(moistureValue);
  Serial.print(",");
  Serial.print(vibrationValue);
  Serial.print(",");
  Serial.print(rainValue);
  Serial.print(",");
  Serial.print(X, 2);
  Serial.print(",");
  Serial.print(Y, 2);
  Serial.print(",");
  Serial.print(Z, 2);
  Serial.print(",");
  Serial.println("\n");

  // ===== Display on LCD =====
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("M:");
  lcd.print(moistureValue);
  lcd.print("% V:");
  lcd.print(vibrationValue);
  lcd.print("% ");

  lcd.setCursor(0, 1);
  lcd.print("R:");
  lcd.print(rainValue);
  delay(1000);
  lcd.clear();
  lcd.print("X:");
  lcd.print(X);
  lcd.print(" Y:");
  lcd.print(Y);
  lcd.setCursor(0, 1);
  lcd.print("Z:");
  lcd.print(Z);

  delay(1000);  // Update every 1 second
}

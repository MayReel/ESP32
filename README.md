```
#include "DHT.h"
#include <Firebase_ESP_Client.h>
#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"
#if defined(ESP32)
#include <WiFi.h>
#elif defined(ESP8266)
#include <ESP8266WiFi.h>
#endif
#include <ESP32Servo.h>
//input ledPin at GPIO_14
#define ledPin 14
//input Temperature and Humidity Sensor at GPIO_18
#define DHTPIN 18
//input fan custom fan speed at GPIO_13
#define PWM 13
//input Light at GPIO_22
#define Relay2 22
//input Servo at GPIO_23
#define SERVO_PIN 23
//Enter your Realtime Database API_Key by following a handbook
#define API_KEY ""
//Enter Your Wifi Username
#define WIFI_SSID "WIFI_NAME"
//Enter Your WiFi Password
#define WIFI_PASSWORD "WIFI_PASSWORD"
//Enter Your Realtime Database link
#define DATABASE_URL ""

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

unsigned long sendDataPrevMillis = 0;
unsigned long wifiCheckPrevMillis = 0;
unsigned long resetPrevMillis = 0;
bool signupOK = false;
#define DHTTYPE DHT22
int pos; //servo position

DHT dht(DHTPIN, DHTTYPE);
Servo doorServo;

void setup() {
  Serial.begin(115200);
  Serial.println(F("DHTxx test!"));
  pinMode(Relay2, OUTPUT);
  pinMode(PWM, OUTPUT);
  doorServo.attach(SERVO_PIN);
  pinMode(ledPin, OUTPUT);
  pinMode(4, INPUT);
  config.api_key = API_KEY;
  checkWiFiStatus();
  config.database_url = DATABASE_URL;

  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("Sign up OK");
    signupOK = true;
  } else {
    Serial.printf("Sign up failed: %s\n", config.signer.signupError.message.c_str());
  }

  config.token_status_callback = tokenStatusCallback;

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  dht.begin();


}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(ledPin, HIGH);
    if (Firebase.ready() && (millis() - sendDataPrevMillis > 2000 || sendDataPrevMillis == 0)) {
      sendDataPrevMillis = millis();

      if (Firebase.RTDB.getString(&fbdo, "Check_Control/Controller")) {
        if (fbdo.dataType() == "string") {
          String Controller = fbdo.stringData();
          Serial.println("Controller: " + Controller);
          readDHTSensor();

          if (Controller == "Lock") {
            updateDoorLockStatus();
          }
          else if (Controller == "Fan") {
            updateFanStatus();
          }
          else if (Controller == "Light") {
            updateLightStatus();
          }
          else if (Controller == "Temp") {
            readDHTSensor();
          }
        }
      }
      else {
        Serial.println(fbdo.errorReason());
      }
    }
  }
  else {
    checkWiFiStatus();
  }
}
void readDHTSensor() {
  float t = dht.readTemperature();
  if (isnan(t)) {
    Serial.println(F("Failed to read from DHT sensor!"));
    return;
  }
  Serial.print(F("Temperature: "));
  Serial.println(t);

  if (Firebase.RTDB.setFloat(&fbdo, "Control_Center/Temperature", t)) {
    Serial.println("Temperature update PASSED");
  } else {
    Serial.println("Temperature update FAILED");
    Serial.println("REASON: " + fbdo.errorReason());
  }
}

void updateFanStatus() {
 
  if (Firebase.RTDB.getInt(&fbdo, "Fan/Current_Status/Power")) {
    int powerStatus = fbdo.intData();
    Serial.println("Fan Power Status: " + String(powerStatus));
    if (powerStatus == 0) {
      analogWrite(PWM, 0);
    } else if (powerStatus == 1) {
      if (Firebase.RTDB.getInt(&fbdo, "Fan/Current_Status/Mode")) {
        int modeStatus = fbdo.intData();
        Serial.println("Fan Mode Status: " + String(modeStatus));
        int pwmValue = modeToPWM(modeStatus);
        analogWrite(PWM, pwmValue);
      } else {
        Serial.println("Failed to get fan mode status");
        Serial.println("REASON: " + fbdo.errorReason());
      }
    }
  } else {
    Serial.println("Failed to get fan power status");
    Serial.println("REASON: " + fbdo.errorReason());
  }

}

int modeToPWM(int mode) {
  switch (mode) {
    case 1: return 85;
    case 2: return 170;
    case 3: return 255;
    default: return 0;
  }
}

void updateLightStatus() {

  if (Firebase.RTDB.getInt(&fbdo, "Light/CurrentStatus")) {
    int lightStatus = fbdo.intData();
    Serial.println("Light Status: " + String(lightStaus));
    digitalWrite(Relay2, lightStatus == 0 ? LOW : HIGH);
  } else {
    Serial.println("Failed to get light status");
    Serial.println("REASON: " + fbdo.errorReason());
  }

}

void updateDoorLockStatus() {
  if (Firebase.RTDB.getInt(&fbdo, "Lock/CurrentStatus")) {
    if (fbdo.dataType() == "int") {
      pos = fbdo.intData();

      doorServo.write(pos);
      Serial.println("Servo position updated: " + String(pos));
    }
  }
  else {
    Serial.println(fbdo.errorReason());
  }
}



void checkWiFiStatus() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    digitalWrite(ledPin, HIGH);
    delay(1000);
    digitalWrite(ledPin, LOW);
    delay(1000);
  }
  Serial.println();
  
  Serial.println("Wi-Fi connected");
  Serial.print("Connected with IP: ");
  Serial.println(WiFi.localIP());
  Serial.println();




}
```

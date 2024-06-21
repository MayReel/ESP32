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

#define ledPin 14
#define DHTPIN 18
#define PWM 13
#define Relay2 22  
#define SERVO_PIN 23  
#define API_KEY ""
#define WIFI_SSID ""
#define WIFI_PASSWORD ""
#define DATABASE_URL ""

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

unsigned long sendDataPrevMillis = 0;
unsigned long wifiCheckPrevMillis = 0;
unsigned long resetPrevMillis = 0;
bool signupOK = false;
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
Servo doorServo;  

void setup() {
    Serial.begin(115200);
    Serial.println(F("DHTxx test!"));

    config.api_key = API_KEY;
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    Serial.print("Connecting to Wi-Fi");
    while (WiFi.status() != WL_CONNECTED) {
        Serial.print(".");
        delay(300);
    }
    Serial.println();
    Serial.print("Connected with IP: ");
    Serial.println(WiFi.localIP());
    Serial.println();

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

    pinMode(Relay2, OUTPUT);
    pinMode(PWM, OUTPUT);
    doorServo.attach(SERVO_PIN);  
    pinMode(ledPin, OUTPUT);  
}

void loop() {
    if (millis() - sendDataPrevMillis > 2000) {
        sendDataPrevMillis = millis();
        readDHTSensor();
        updateFanStatus();
        updateLightStatus();
        updateDoorLockStatus();
    }
    
    if (millis() - wifiCheckPrevMillis > 1000) {  
        wifiCheckPrevMillis = millis();
        checkWiFiStatus();
    }

    if (millis() - resetPrevMillis > 60000) {  
        resetPrevMillis = millis();
        ESP.restart();  // Reset the ESP32/ESP8266
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

    if (Firebase.ready() && signupOK) {
        if (Firebase.RTDB.setFloat(&fbdo, "Control_Center/Temperature", t)) {
            Serial.println("Temperature update PASSED");
        } else {
            Serial.println("Temperature update FAILED");
            Serial.println("REASON: " + fbdo.errorReason());
        }
    }
}

void updateFanStatus() {
    if (Firebase.ready() && signupOK) {
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
    if (Firebase.ready() && signupOK) {
        if (Firebase.RTDB.getInt(&fbdo, "Light/CurrentStatus")) {
            int lightStatus = fbdo.intData();
            Serial.println("Light Status: " + String(lightStatus));
            digitalWrite(Relay2, lightStatus == 0 ? LOW : HIGH);
        } else {
            Serial.println("Failed to get light status");
            Serial.println("REASON: " + fbdo.errorReason());
        }
    }
}

void updateDoorLockStatus() {
    if (Firebase.ready() && signupOK) {
        if (Firebase.RTDB.getInt(&fbdo, "Lock/CurrentStatus")) {
            int LockStatus = fbdo.intData();
            Serial.println("Lock Status: " + String(LockStatus));
            doorServo.write(LockStatus == 0 ? 90 : 0);  
        } else {
            Serial.println("Failed to get Lock status");
            Serial.println("REASON: " + fbdo.errorReason());
        }
    }
}

void checkWiFiStatus() {
    if (WiFi.status() != WL_CONNECTED) {
        digitalWrite(ledPin, LOW);  
        Serial.println("Wi-Fi disconnected");
    } else {
        digitalWrite(ledPin, HIGH);  
        Serial.println("Wi-Fi connected");
    }
}
```

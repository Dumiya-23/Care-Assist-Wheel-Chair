#include <ESP8266WiFi.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Servo.h>

const int GYRO_ADDRESS = 0x68;
const int SERVO_PIN = 9;

Adafruit_MPU6050 mpu;
Servo servo;

void setup() {
    Serial.begin(115200);
    Wire.begin();
    mpu.initialize();
    servo.attach(SERVO_PIN);
}

void loop() {
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp);

    // Calculate angle from gyroscope data (adjust as needed)
    float angle = g.z * 0.0174532925; // Convert to radians

    // Determine servo position based on angle
    int servoPosition = map(angle, -15, 15, 0, 180);

    // Set servo position to maintain balance
    servo.write(servoPosition);

    // Print angle and servo position for debugging
    Serial.print("Angle: ");
    Serial.print(angle);
    Serial.print(" Servo: ");
    Serial.println(servoPosition);
}


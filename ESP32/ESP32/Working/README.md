# ESP32 Working Codes 
Working ESP32 programs for AIRA
#include <Arduino.h>
#include <ESP32Servo.h>

const int SERVO_PIN = 33;

const int IR_FRONT_PIN = 36; // VP
const int IR_LEFT_PIN  = 39; // VN
const int IR_RIGHT_PIN = 34; // D34

const int MOTOR_ENA = 18;
const int MOTOR_IN1 = 19;
const int MOTOR_IN2 = 21;

Servo headServo;

bool motorArmed = false;
int motorSpeed = 0;

bool obstacleDetected(int pin) {
  return digitalRead(pin) == LOW;
}

void motorStop() {
  analogWrite(MOTOR_ENA, 0);
  digitalWrite(MOTOR_IN1, LOW);
  digitalWrite(MOTOR_IN2, LOW);
  motorSpeed = 0;
  Serial.println("MOTOR: STOP");
}

void motorForward(int speedValue) {
  if (!motorArmed) {
    Serial.println("MOTOR BLOCKED: not armed");
    return;
  }

  if (obstacleDetected(IR_FRONT_PIN)) {
    Serial.println("MOTOR BLOCKED: front obstacle");
    motorStop();
    return;
  }

  digitalWrite(MOTOR_IN1, HIGH);
  digitalWrite(MOTOR_IN2, LOW);
  analogWrite(MOTOR_ENA, speedValue);
  motorSpeed = speedValue;
  Serial.print("MOTOR: FORWARD ");
  Serial.println(speedValue);
}

void motorReverse(int speedValue) {
  if (!motorArmed) {
    Serial.println("MOTOR BLOCKED: not armed");
    return;
  }

  digitalWrite(MOTOR_IN1, LOW);
  digitalWrite(MOTOR_IN2, HIGH);
  analogWrite(MOTOR_ENA, speedValue);
  motorSpeed = -speedValue;
  Serial.print("MOTOR: REVERSE ");
  Serial.println(speedValue);
}

void printStatus() {
  bool front = obstacleDetected(IR_FRONT_PIN);
  bool left  = obstacleDetected(IR_LEFT_PIN);
  bool right = obstacleDetected(IR_RIGHT_PIN);

  Serial.print("IR Front=");
  Serial.print(front ? "OBSTACLE" : "CLEAR");
  Serial.print(" | Left=");
  Serial.print(left ? "OBSTACLE" : "CLEAR");
  Serial.print(" | Right=");
  Serial.print(right ? "OBSTACLE" : "CLEAR");
  Serial.print(" | Armed=");
  Serial.print(motorArmed ? "YES" : "NO");
  Serial.print(" | MotorSpeed=");
  Serial.println(motorSpeed);
}

void scanHead() {
  Serial.println("HEAD: SCAN");
  for (int angle = 30; angle <= 150; angle += 5) {
    headServo.write(angle);
    delay(40);
  }
  for (int angle = 150; angle >= 30; angle -= 5) {
    headServo.write(angle);
    delay(40);
  }
  headServo.write(90);
  Serial.println("HEAD: CENTER");
}

void handleCommand(char cmd) {
  if (cmd == 'A' || cmd == 'a') {
    motorArmed = true;
    Serial.println("MOTOR: ARMED");
  }

  if (cmd == 'D' || cmd == 'd') {
    motorArmed = false;
    motorStop();
    Serial.println("MOTOR: DISARMED");
  }

  if (cmd == 'F' || cmd == 'f') {
    motorForward(150);
  }

  if (cmd == 'B' || cmd == 'b') {
    motorReverse(150);
  }

  if (cmd == 'X' || cmd == 'x') {
    motorStop();
    Serial.println("EMERGENCY STOP");
  }

  if (cmd == 'L' || cmd == 'l') {
    headServo.write(150);
    Serial.println("HEAD: LEFT");
  }

  if (cmd == 'C' || cmd == 'c') {
    headServo.write(90);
    Serial.println("HEAD: CENTER");
  }

  if (cmd == 'R' || cmd == 'r') {
    headServo.write(30);
    Serial.println("HEAD: RIGHT");
  }

  if (cmd == 'S' || cmd == 's') {
    scanHead();
  }

  if (cmd == 'P' || cmd == 'p') {
    printStatus();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  pinMode(IR_FRONT_PIN, INPUT);
  pinMode(IR_LEFT_PIN, INPUT);
  pinMode(IR_RIGHT_PIN, INPUT);

  pinMode(MOTOR_ENA, OUTPUT);
  pinMode(MOTOR_IN1, OUTPUT);
  pinMode(MOTOR_IN2, OUTPUT);

  motorStop();

  headServo.setPeriodHertz(50);
  headServo.attach(SERVO_PIN, 500, 2400);
  headServo.write(90);

  Serial.println("AIRA ESP32 V1 firmware ready");
  Serial.println("Commands:");
  Serial.println("A = arm motor");
  Serial.println("D = disarm motor");
  Serial.println("F = forward");
  Serial.println("B = reverse");
  Serial.println("X = emergency stop");
  Serial.println("L/C/R = head left/center/right");
  Serial.println("S = scan head");
  Serial.println("P = print status");
}

void loop() {
  if (obstacleDetected(IR_FRONT_PIN) && motorSpeed > 0) {
    Serial.println("SAFETY: front obstacle detected, stopping");
    motorStop();
  }

  if (Serial.available()) {
    char cmd = Serial.read();
    handleCommand(cmd);
  }

  static unsigned long lastStatus = 0;
  if (millis() - lastStatus > 1000) {
    printStatus();
    lastStatus = millis();
  }
}


working for the following wiring
Servo orange -> D33
Servo red    -> VIN/5V
Servo brown  -> GND

Front IR OUT -> VP/GPIO36
Left IR OUT  -> VN/GPIO39
Right IR OUT -> D34/GPIO34
All IR VCC   -> 3V3
All IR GND   -> GND

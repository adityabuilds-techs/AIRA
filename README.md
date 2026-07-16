# 🤖 AIRA

AIRA is an AI-powered robotic assistant being developed by Aditya.

## 🚀 Features

- Voice Assistant
- Face Recognition
- Obstacle Avoidance
- Camera Vision
- Raspberry Pi + ESP32 Communication
- Touch Display Interface
- AI Interaction

## 🛠 Hardware

- Raspberry Pi 4
- ESP32
- Raspi Camera Rev 1.3
- Raspi Display
- HC-SR04 Ultrasonic Sensor
- TT Motors
- L298N Motor Driver
- USB Microphone
- Speaker
- I.R sensors

## 📈 Project Status

🚧 Under Development

## 📌 Future Goals

- Autonomous Navigation
- Smart Home Control
- Offline AI Assistant














  # AIRA Raspberry Pi Bridge

This is the first Raspberry Pi milestone for AIRA V1.

It creates a small web dashboard on the Pi and sends commands to the ESP32 over
USB serial. It does not need the 3.5 inch display, camera, or any Python
packages.

## Hardware Needed

- Raspberry Pi powered and connected to Wi-Fi
- ESP32 connected to Raspberry Pi by USB
- ESP32 already flashed with the AIRA ESP32 safety firmware

## Run On Raspberry Pi

Create the folder:

```bash
mkdir -p ~/aira_v1
cd ~/aira_v1
```

Create the file:

```bash
nano aira_bridge.py
```

Paste the contents of `aira_bridge.py`, save, then run:

```bash
python3 aira_bridge.py
```

If the ESP32 is not on `/dev/ttyUSB0`, check:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

Then run with the correct port:

```bash
AIRA_SERIAL_PORT=/dev/ttyACM0 python3 aira_bridge.py
```

## Open From Phone

Find the Pi IP:

```bash
hostname -I
```

Open this on your phone browser:

```text
http://PI_IP_HERE:8080
```

Example:

```text
http://192.168.1.44:8080
```

## Dashboard Commands

- `ARM`: allow motor movement commands
- `FORWARD`: send `F`
- `BACK`: send `B`
- `STOP`: emergency stop, send `X`
- `DISARM`: disable motor commands
- `STATUS`: request ESP32 status

## Notes

For the first Pi test, keep motor battery power disconnected. The web dashboard
should still show ESP32 serial logs and command responses.




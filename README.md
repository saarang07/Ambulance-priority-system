# Ambulance Priority System

An Arduino-based prototype that gives an approaching ambulance priority at a junction. The ambulance unit sends a LoRa message when its button is pressed. The junction unit receives that message, switches the traffic LEDs to green, raises a servo-operated barrier, and uses an ultrasonic sensor to detect a vehicle passing before restoring the normal red-light state.

> **Prototype note:** This project demonstrates a model junction. It is not a certified traffic-control or road-safety system and should not control a real road intersection.

## How it works

1. The ambulance unit monitors a push button using the Arduino's internal pull-up resistor.
2. When the button is pressed, it transmits `AMBULANCE` over LoRa at 433 MHz.
3. The junction unit receives the message, turns on its green LED, turns off its red LED, and rotates the servo to raise the barrier.
4. When the ultrasonic sensor reports an object less than 30 cm away, the junction waits three seconds, returns to red, lowers the barrier, and exits emergency mode.

## Hardware

### Ambulance unit

- Arduino board with SPI support (the pin examples below assume an Uno/Nano-style board)
- 433 MHz LoRa transceiver compatible with the `LoRa` Arduino library
- Push button
- Jumper wires and suitable power source

### Junction unit

- Arduino board with SPI support (the pin examples below assume an Uno/Nano-style board)
- Matching 433 MHz LoRa transceiver
- Red and green LEDs with current-limiting resistors
- HC-SR04-compatible ultrasonic distance sensor
- Servo motor for the model barrier
- Jumper wires and suitable power source

Both LoRa radios must use compatible hardware, antennae, frequency, and radio settings. Check the radio module's voltage requirements and use appropriate level shifting where needed; many LoRa modules use 3.3 V logic and can be damaged by 5 V signals.

## Wiring

The LoRa connections use the Arduino SPI bus. For Uno/Nano-style boards, the SPI pins are MOSI 11, MISO 12, and SCK 13. Other Arduino boards may use different SPI pins.

### Ambulance unit

| Component | Arduino pin |
|---|---:|
| LoRa NSS/CS | 10 |
| LoRa RESET | 9 |
| LoRa DIO0 | 2 |
| LoRa MOSI / MISO / SCK | 11 / 12 / 13 (Uno/Nano SPI) |
| Push button | 7 to GND |

The button is configured as `INPUT_PULLUP`, so it should connect between pin 7 and GND; no external pull-up resistor is required.

### Junction unit

| Component | Arduino pin |
|---|---:|
| LoRa NSS/CS | 10 |
| LoRa RESET | 9 |
| LoRa DIO0 | 8 |
| LoRa MOSI / MISO / SCK | 11 / 12 / 13 (Uno/Nano SPI) |
| Red LED (through resistor) | 2 |
| Green LED (through resistor) | 3 |
| Ultrasonic TRIG | 4 |
| Ultrasonic ECHO | 5 |
| Servo signal | 6 |

Connect the sensor and servo power and grounds according to their specifications. For a servo, a separate regulated supply may be needed; connect its ground to the Arduino ground. Confirm the LoRa module's pinout from its manufacturer before wiring.

## Software setup

1. Install the [Arduino IDE](https://www.arduino.cc/en/software).
2. Install the **LoRa** library by Sandeep Mistry from the Arduino Library Manager.
3. Open each sketch below in a separate Arduino IDE window or save each code block as its own `.ino` file.
4. Select the correct board and port, then upload the corresponding sketch to each Arduino.
5. Open the Serial Monitor at **9600 baud** to view status messages.
6. Power both units and press the ambulance button. On the junction unit, check for `AMBULANCE` received and the green-light status.

The sketches use `LoRa.begin(433E6)`. Use 433 MHz only where permitted for your location and with radio modules configured for that band.

## Ambulance unit code

```cpp
#include <SPI.h>
#include <LoRa.h>

#define LORA_SS    10
#define LORA_RST   9
#define LORA_DIO0  2

#define BUTTON_PIN 7

bool lastButtonState = HIGH;

void setup() {
  Serial.begin(9600);

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  LoRa.setPins(LORA_SS, LORA_RST, LORA_DIO0);

  Serial.println("Starting Ambulance LoRa...");

  if (!LoRa.begin(433E6)) {
    Serial.println("LoRa initialization FAILED!");
    while (true);
  }

  Serial.println("LoRa initialized successfully.");
  Serial.println("Ambulance system READY.");
}

void loop() {

  bool buttonState = digitalRead(BUTTON_PIN);

  if (lastButtonState == HIGH && buttonState == LOW) {

    Serial.println("AMBULANCE BUTTON PRESSED");

    LoRa.beginPacket();
    LoRa.print("AMBULANCE");
    LoRa.endPacket();

    Serial.println("Emergency signal SENT");

    delay(500);
  }

  lastButtonState = buttonState;
}
```

## Junction unit code

```cpp
#include <SPI.h>
#include <LoRa.h>
#include <Servo.h>

// ---------------- LoRa pins ----------------
#define LORA_SS    10
#define LORA_RST   9
#define LORA_DIO0  8

// ---------------- Traffic LEDs ----------------
#define RED_LED    2
#define GREEN_LED  3

// ---------------- Ultrasonic sensor ----------------
#define TRIG_PIN   4
#define ECHO_PIN   5

// ---------------- Servo ----------------
#define SERVO_PIN  6

Servo barrierServo;

// System state
bool emergencyMode = false;

void setup() {

  Serial.begin(9600);

  // Traffic LEDs
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  // Ultrasonic
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Servo
  barrierServo.attach(SERVO_PIN);

  // Initial condition
  digitalWrite(RED_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);

  barrierServo.write(0);

  // LoRa
  LoRa.setPins(LORA_SS, LORA_RST, LORA_DIO0);

  Serial.println("Starting Junction LoRa...");

  if (!LoRa.begin(433E6)) {

    Serial.println("LoRa initialization FAILED!");

    while (true) {
      delay(1000);
    }
  }

  Serial.println("LoRa initialized successfully.");
  Serial.println("Junction system READY.");
}

void loop() {

  // Check for LoRa message
  int packetSize = LoRa.parsePacket();

  if (packetSize) {

    String receivedMessage = "";

    while (LoRa.available()) {
      receivedMessage += (char)LoRa.read();
    }

    Serial.print("Received: ");
    Serial.println(receivedMessage);

    // Check emergency message
    if (receivedMessage == "AMBULANCE") {

      Serial.println("AMBULANCE DETECTED!");

      emergencyMode = true;

      // Give ambulance priority
      digitalWrite(RED_LED, LOW);
      digitalWrite(GREEN_LED, HIGH);

      // Raise barrier
      barrierServo.write(90);

      Serial.println("GREEN LIGHT - AMBULANCE PRIORITY");
    }
  }

  // If emergency mode is active
  if (emergencyMode) {

    long distance = getDistance();

    Serial.print("Distance: ");
    Serial.print(distance);
    Serial.println(" cm");

    // Detect ambulance/vehicle passing
    if (distance > 0 && distance < 30) {

      Serial.println("Vehicle detected by ultrasonic sensor.");

      delay(3000);

      // Return to normal traffic
      digitalWrite(GREEN_LED, LOW);
      digitalWrite(RED_LED, HIGH);

      // Lower barrier
      barrierServo.write(0);

      emergencyMode = false;

      Serial.println("Normal traffic resumed.");

      delay(1000);
    }
  }

  delay(50);
}

// ---------------- Ultrasonic function ----------------

long getDistance() {

  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);

  if (duration == 0) {
    return -1;
  }

  long distance = duration * 0.034 / 2;

  return distance;
}
```

## Current behavior and limitations

- The ambulance sketch sends one packet for each detected button press. It does not keep transmitting while the button remains pressed.
- The junction considers any measured object closer than 30 cm a passing vehicle. Mount and test the sensor so ordinary nearby objects do not trigger the reset.
- The junction waits three seconds after detection and then returns to normal, without checking that the sensor is clear or receiving a cancellation message.
- The sketches do not include acknowledgements, retries, sender authentication, or protection against missed or duplicate packets.
- LED colors and the barrier position are controlled by the code; adapt them to the physical model and verify behavior before demonstration.

## Repository layout

```text
ambulance-priority-system/
├── README.md
├── ambulance/
│   └── ambulance.ino
└── junction/
    └── junction.ino
```

Save the ambulance and junction sketches in the folders shown above to match this suggested layout.

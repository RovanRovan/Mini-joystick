# Mini-joystick
It is made just for fun, and can be a fun add-on for something like a Sayodevice or other devices if you dont want to switch between things all the time, or if you are traveling.


A small USB mouse-style joystick made using a Pimoroni Trackball Breakout and a Waveshare RP2040-Zero.

The Trackball acts as the mouse controller, while the RP2040-Zero converts its movement and click into a USB mouse that can be used with a computer.

## Features

* USB mouse control
* Trackball movement
* Trackball click
* RGBW Trackball LED
* Adjustable sensitivity
* Adjustable acceleration
* Adjustable smoothing
* Adjustable deadzone
* X/Y movement inversion
* Simple customization
* Only four wires between the two boards

## Parts

* Pimoroni Trackball Breakout
* Waveshare RP2040-Zero
* 4 short pieces of wire
* USB-C data cable
* Computer for programming

## Wiring

Connect the four wires like this:

| Pimoroni Trackball | RP2040-Zero |
| ------------------ | ----------- |
| 3V3                | 3V3         |
| GND                | GND         |
| SDA                | GP4         |
| SCL                | GP5         |

The Trackball doesn't need its own USB connection.

The USB-C cable connects to the RP2040-Zero.

## Software

This project is programmed using Arduino IDE.

You will need:

1. Arduino IDE
2. Raspberry Pi RP2040 board support
3. The Pimoroni Trackball Arduino library

After installing these, open:

`Arduino/TrackballMouse/TrackballMouse.ino`

Then upload it to the RP2040-Zero.

## Customization

All of the main settings are located at the **bottom of the Arduino code**.

You do not need to change the rest of the program.

For example:

```cpp
float SENSITIVITY = 2.0;
float ACCELERATION = 1.2;
int SMOOTHING = 10;
int DEADZONE = 0;

bool INVERT_X = false;
bool INVERT_Y = false;
```

### Sensitivity

Controls how far the mouse moves when you move the Trackball.

```cpp
float SENSITIVITY = 2.0;
```

Higher number = faster mouse movement.

### Acceleration

Controls how much faster the cursor moves when the Trackball is moved quickly.

```cpp
float ACCELERATION = 1.2;
```

### Smoothing

Makes movement less abrupt.

```cpp
int SMOOTHING = 10;
```

### Invert X/Y

Reverse the direction of horizontal or vertical movement.

```cpp
bool INVERT_X = false;
bool INVERT_Y = false;
```

Change `false` to `true` to reverse a direction.

### LED

The Trackball has an RGBW LED that can also be customized.

```cpp
int LED_RED   = 30;
int LED_GREEN = 80;
int LED_BLUE  = 255;
int LED_WHITE = 0;
```

Each value ranges from `0` to `255`.

## How it works

```text
Your finger
     ↓
Pimoroni Trackball
     ↓
    I²C
     ↓
RP2040-Zero
     ↓
   USB-C
     ↓
Computer
```

The Pimoroni Trackball detects movement and clicking. The RP2040-Zero reads that information and sends it to the computer as a USB mouse.

## Project Status

This project is currently being developed and tested.

More features and improvements may be added in the future.



## Joystick Code
/*
   ============================================================
                 PIMORONI TRACKBALL USB MOUSE
   ============================================================

   Hardware:
   - Waveshare RP2040-Zero
   - Pimoroni Trackball Breakout

   Connections:
   Trackball 3V3 -> RP2040 3V3
   Trackball GND -> RP2040 GND
   Trackball SDA -> RP2040 GP4
   Trackball SCL -> RP2040 GP5

   Only FOUR wires are required.

   The RP2040 appears to the computer as a normal USB mouse.

   ============================================================
*/


#include <Arduino.h>
#include <Wire.h>
#include <Mouse.h>
#include <Pimoroni_Trackball.h>


// ============================================================
//                    HARDWARE SETUP
// ============================================================

#define I2C_SDA 4
#define I2C_SCL 5

Pimoroni_Trackball trackball(
  Pimoroni_Trackball::I2C_ADDRESS,
  &Wire
);


// ============================================================
//                    MOVEMENT VARIABLES
// ============================================================

float smoothX = 0;
float smoothY = 0;

bool previousButton = false;


// ============================================================
//                         SETUP
// ============================================================

void setup() {

  // -------------------------
  // Start I2C
  // -------------------------

  Wire.setSDA(I2C_SDA);
  Wire.setSCL(I2C_SCL);

  Wire.begin();

  // The Trackball works reliably with 100 kHz I2C.
  Wire.setClock(100000);


  // -------------------------
  // Start USB mouse
  // -------------------------

  Mouse.begin();


  // -------------------------
  // Start Trackball
  // -------------------------

  if (!trackball.begin()) {

    // If the Trackball isn't detected,
    // stay here and flash the built-in LED.

    pinMode(LED_BUILTIN, OUTPUT);

    while (true) {

      digitalWrite(LED_BUILTIN, HIGH);
      delay(150);

      digitalWrite(LED_BUILTIN, LOW);
      delay(150);
    }
  }


  // -------------------------
  // Set Trackball LED
  // -------------------------

  setTrackballLED(
    LED_RED,
    LED_GREEN,
    LED_BLUE,
    LED_WHITE
  );


  delay(100);
}


// ============================================================
//                          LOOP
// ============================================================

void loop() {

  updateTrackball();

  delay(UPDATE_DELAY_MS);
}


// ============================================================
//                  TRACKBALL PROCESSING
// ============================================================

void updateTrackball() {

  uint8_t rawX;
  uint8_t rawY;

  // Get movement from the Pimoroni driver.
  trackball.getMotion(rawX, rawY);


  // ----------------------------------------------------------
  // Convert the unsigned 8-bit values into signed movement.
  //
  // The driver calculates:
  // X = right - left
  // Y = up - down
  // ----------------------------------------------------------

  int x = (int8_t)rawX;
  int y = (int8_t)rawY;


  // ----------------------------------------------------------
  // DEADZONE
  // ----------------------------------------------------------

  if (abs(x) <= DEADZONE) {
    x = 0;
  }

  if (abs(y) <= DEADZONE) {
    y = 0;
  }


  // ----------------------------------------------------------
  // SENSITIVITY
  // ----------------------------------------------------------

  float moveX = x * SENSITIVITY;
  float moveY = y * SENSITIVITY;


  // ----------------------------------------------------------
  // ACCELERATION
  // ----------------------------------------------------------

  if (ACCELERATION != 1.0) {

    if (moveX != 0) {

      float direction =
        moveX > 0 ? 1.0 : -1.0;

      moveX =
        direction *
        pow(abs(moveX), ACCELERATION);
    }


    if (moveY != 0) {

      float direction =
        moveY > 0 ? 1.0 : -1.0;

      moveY =
        direction *
        pow(abs(moveY), ACCELERATION);
    }
  }


  // ----------------------------------------------------------
  // INVERT X
  // ----------------------------------------------------------

  if (INVERT_X) {
    moveX = -moveX;
  }


  // ----------------------------------------------------------
  // INVERT Y
  // ----------------------------------------------------------

  if (INVERT_Y) {
    moveY = -moveY;
  }


  // ----------------------------------------------------------
  // SMOOTHING
  // ----------------------------------------------------------

  float smoothing =
    SMOOTHING / 100.0;

  float response =
    1.0 - smoothing;


  smoothX =
    smoothX * smoothing +
    moveX * response;

  smoothY =
    smoothY * smoothing +
    moveY * response;


  // ----------------------------------------------------------
  // SEND MOVEMENT TO COMPUTER
  // ----------------------------------------------------------

  int finalX =
    round(smoothX * MOUSE_MULTIPLIER);

  int finalY =
    round(smoothY * MOUSE_MULTIPLIER);


  if (finalX != 0 || finalY != 0) {

    Mouse.move(
      finalX,
      finalY,
      0
    );
  }


  // ----------------------------------------------------------
  // CLICK
  // ----------------------------------------------------------

  bool buttonPressed =
    trackball.getButton() != 0;


  if (CLICK_ENABLED) {

    // Button was just pressed
    if (buttonPressed && !previousButton) {

      Mouse.press(MOUSE_LEFT);
    }


    // Button was just released
    if (!buttonPressed && previousButton) {

      Mouse.release(MOUSE_LEFT);
    }
  }


  previousButton =
    buttonPressed;


  // ----------------------------------------------------------
  // STOP SMOOTHING FROM DRIFTING
  // ----------------------------------------------------------

  if (x == 0) {
    smoothX *= 0.70;
  }

  if (y == 0) {
    smoothY *= 0.70;
  }
}


// ============================================================
//                     TRACKBALL LED
// ============================================================
//
// The library's LED function has an issue in some versions,
// so this sketch writes the four LED registers directly.
//
// The Trackball's official register layout is:
// 0x00 = Red
// 0x01 = Green
// 0x02 = Blue
// 0x03 = White
//
// ============================================================

void setTrackballLED(
  uint8_t red,
  uint8_t green,
  uint8_t blue,
  uint8_t white
) {

  Wire.beginTransmission(
    Pimoroni_Trackball::I2C_ADDRESS
  );

  Wire.write(0x00);

  Wire.write(red);
  Wire.write(green);
  Wire.write(blue);
  Wire.write(white);

  Wire.endTransmission();
}


// ============================================================
// ============================================================
//                     CUSTOMIZE HERE
// ============================================================
// ============================================================
//
// YOU ONLY NEED TO CHANGE THE SETTINGS BELOW.
//
// Everything above this section runs the device.
//
// ============================================================


// ------------------------------------------------------------
//                    MOUSE SENSITIVITY
// ------------------------------------------------------------
//
// 1.0 = normal
// 2.0 = twice as fast
// 3.0 = three times as fast
//
// Recommended range:
// 0.5 - 4.0
//

float SENSITIVITY = 1.5;


// ------------------------------------------------------------
//                     ACCELERATION
// ------------------------------------------------------------
//
// 1.0 = no acceleration
// 1.2 = slight acceleration
// 1.5 = noticeable acceleration
// 2.0 = strong acceleration
//
// Recommended:
// 1.0 - 1.5
//

float ACCELERATION = 1.0;


// ------------------------------------------------------------
//                      SMOOTHING
// ------------------------------------------------------------
//
// 0 = no smoothing
// 10 = slight smoothing
// 25 = moderate smoothing
// 50 = strong smoothing
// 75 = very strong smoothing
//
// Recommended:
// 5 - 25
//

int SMOOTHING = 15;


// ------------------------------------------------------------
//                       DEADZONE
// ------------------------------------------------------------
//
// Ignores very tiny movements.
//
// 0 = everything is detected
// 1 = tiny movements ignored
// 2 = more ignored
// 3+ = increasingly large deadzone
//
// Recommended:
// 0 - 2
//

int DEADZONE = 0;


// ------------------------------------------------------------
//                      INVERT X
// ------------------------------------------------------------
//
// false = normal
// true  = reverse left/right
//

bool INVERT_X = false;


// ------------------------------------------------------------
//                      INVERT Y
// ------------------------------------------------------------
//
// false = normal
// true  = reverse up/down
//

bool INVERT_Y = false;


// ------------------------------------------------------------
//                    MOUSE MULTIPLIER
// ------------------------------------------------------------
//
// Extra multiplier applied after all other processing.
//
// 1.0 = normal
// 2.0 = twice as much movement
// 0.5 = half as much movement
//
// Usually leave this at 1.0.
//

float MOUSE_MULTIPLIER = 1.0;


// ============================================================
//                       CLICK SETTINGS
// ============================================================
//
// true  = Trackball click works
// false = Trackball click disabled
//

bool CLICK_ENABLED = true;


// ============================================================
//                        LED COLOR
// ============================================================
//
// Each value is from 0 to 255.
//
// RGB examples:
//
// Blue:
// R = 0
// G = 0
// B = 255
//
// Red:
// R = 255
// G = 0
// B = 0
//
// Green:
// R = 0
// G = 255
// B = 0
//
// Purple:
// R = 255
// G = 0
// B = 255
//
// White:
// R = 0
// G = 0
// B = 0
// W = 255
//

int LED_RED   = 30;
int LED_GREEN = 80;
int LED_BLUE  = 255;
int LED_WHITE = 0;


// ============================================================
//                    UPDATE SPEED
// ============================================================
//
// How quickly the Trackball is checked.
//
// Smaller number = slightly faster response.
//
// 1 = very fast
// 2 = recommended
// 5 = slower
//

int UPDATE_DELAY_MS = 2;


// ============================================================
//                       END SETTINGS
// ============================================================

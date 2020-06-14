---
layout: default
title: LED Setup
nav_order: 4
permalink: /led-setup/
---

# Setting Up the LED Control

Let's start by setting up what we need to handle the LED we've wired to the ESP32 on the breadboard. As a reminder, the LED is connected to the **GPIO 26** pin of the ESP32.

Open the `main.cpp` file in the `esp32-rc-ws` directory of **your** project and add the following lines to it:

```cpp
#include <Arduino.h>

// ----------------------------------------------------------------------------
// Definition of macros
// ----------------------------------------------------------------------------

#define LED_PIN 26

// ----------------------------------------------------------------------------
// Definition of the Led component
// ----------------------------------------------------------------------------

struct Led {
    // state variables
    uint8_t pin;
    bool    on;
};
```

A simple data structure allows us to associate the control pin of the LED with its state. The `on` property is used here to define the state of the LED and determine whether it is on (`true`) or off (`false`).

This data structure is used to define a model, i.e. an abstract representation of an LED. It can be thought of as a construction plan. We can then define an instance of this model, i.e. an object built according to the construction plan described by the model:

```cpp
// ----------------------------------------------------------------------------
// Definition of global variables
// ----------------------------------------------------------------------------

Led led = { LED_PIN, false };
```

Once the LED has been defined, its control pin must be initialized as an output pin:

```cpp
// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

void setup() {
    pinMode(led.pin, OUTPUT);
}
```

In addition to defining what an object is made of (its structural properties), we can also define how it can act on its environment or on itself (its behavioural properties). To do this, we only have to define functions that describe its ability to act. We will rather talk about *methods*. We are very close to the paradigm of object-oriented programming, although this is not quite the case. I won’t go into these details here, so as not to complicate my point.

Well, we can define an `update()` method which, depending on its state described by its `on` property, sends the right signal to its control pin to turn on (`HIGH`) or off (`LOW`):

```cpp
// ----------------------------------------------------------------------------
// Definition of the Led component
// ----------------------------------------------------------------------------

struct Led {
    // state variables
    uint8_t pin;
    bool    on;

    // methods
    void update() {
        digitalWrite(pin, on ? HIGH : LOW);
    }
};
```

Now, to simply flash the LED at regular intervals (let's say one second), just add this piece of code to the `loop()` function:

```cpp
// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    led.on = millis() % 1000 < 50;
    led.update();
}
```

You can now compile and upload the program. And here's what happens:

<div class="video-wrapper shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo-led.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

You can update the baseline project by going to the `remote-control-with-websocket` directory and executing the following `git` command:

```
git checkout v0.1
```

You can also download the updated `main.cpp` file and replace yours in your own project:

<div class="files">
    <a class="icon" href="{% include github-download.html file="/v0.1/src/main.cpp" %}" download="main.cpp">
        <img width="64" src="{{ 'images/file-cpp-icon-178x192.png' | relative_url }}">
        <p class="filename">main.cpp</p>
    </a>
</div>


Here is the complete code for this chapter:

```cpp
/**
 * ----------------------------------------------------------------------------
 * ESP32 Remote Control with WebSocket
 * ----------------------------------------------------------------------------
 * © 2020 Stéphane Calderoni
 * ----------------------------------------------------------------------------
 */

#include <Arduino.h>

// ----------------------------------------------------------------------------
// Definition of macros
// ----------------------------------------------------------------------------

#define LED_PIN 26

// ----------------------------------------------------------------------------
// Definition of the Led component
// ----------------------------------------------------------------------------

struct Led {
    // state variables
    uint8_t pin;
    bool    on;

    // methods
    void update() {
        digitalWrite(pin, on ? HIGH : LOW);
    }
};

// ----------------------------------------------------------------------------
// Definition of global variables
// ----------------------------------------------------------------------------

Led led = { LED_PIN, false };

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

void setup() {
    pinMode(led.pin, OUTPUT);
}

// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    led.on = millis() % 1000 < 50;
    led.update();
}
```


[PlatformIO IDE]({{ '/platformio-ide/' | relative_url }}){: .btn }
[Button Setup]({{ '/button-setup/' | relative_url }}){: .btn .btn-purple }
{: .navigator }
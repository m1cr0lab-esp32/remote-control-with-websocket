---
layout: default
title: Button Setup
nav_order: 5
permalink: /button-setup/
---

# Setting Up the Push Button Control

Now that the LED is functional, we'll take care of the push button. As a reminder, the button is connected to **GPIO 22** of the ESP32, which is also associated with a pull-up resistor to prevent the signal from floating.


## The bouncing problem

When using a push button, you must be aware that it is an imperfect mechanical device and very unstable given the time scale at which electrical phenomena occur.

> To you and me, when you press a pushbutton, it goes from an open circuit to a closed one, and that single transition defines the button “press”. But the reason that it’s so clear-cut to us is because we perceive things pretty slowly. To a microcontroller, which is able to test a button’s state as often as a few million times per second, it’s not on-off at all.
> 
> When you physically press a normal pushbutton, two pieces of metal come into contact with each other. If these two tiny sheets of metal aren’t perfectly flat or perfectly aligned (and they’re not) then they can make and break contact a few times before being firmly enough squished together that they’re always conducting. To a microcontroller the button appears to be pressed many times for extremely short durations, when you think you’ve just pressed it once. Debouncing is all about making sure that you and the microcontroller agree about when a button push or release event happened.
> 
> ![Problem of Bouncing][bouncing]{: width="100%" .with-caption .shadow }
> Credits: Elliot Williams on [Hackaday][hackaday]
> {: .caption }

Contact bounce can be entirely eliminated with a little bit of hardware. You will find many examples that deal with this issue on the Net. But what I'm going to show you here allows you to simply solve the problem with a bit of code.


## Implementation of a Button component

Let's start by setting the pin for reading the status of the button:

```cpp
// ----------------------------------------------------------------------------
// Definition of macros
// ----------------------------------------------------------------------------

#define LED_PIN 26
#define BTN_PIN 22
```

As you can see from the figure above, the bounce problem occurs over a short period of time during which the signal is unstable. We will therefore have to inhibit the signal over a period that covers this instability. To do this, let's define the waiting time beyond which we will be able to take the signal into account without risking a reading error:

```cpp
// ----------------------------------------------------------------------------
// Definition of global constants
// ----------------------------------------------------------------------------

const uint8_t DEBOUNCE_DELAY = 10; // in milliseconds
```

This delay is set empirically and depends on your button. If you observe any instability later, you can simply extend this time.

As for the `Led` component, we are going to create a data structure to handle the button:

```cpp
// ----------------------------------------------------------------------------
// Definition of the Button component
// ----------------------------------------------------------------------------

struct Button {
    // state variables
    uint8_t  pin;
    bool     lastReading;
    uint32_t lastDebounceTime;
    uint16_t state;
};
```

- `pin` defines the input pin to which the button is connected
- `lastReading` defines the last read value of the signal (`HIGH` or `LOW`)
- `lastDebounceTime` defines the last time, in milliseconds, at which the last reading was performed
- `state` is a 16-bit integer that defines the state of the button as follows:

![Button States][states]{: width="512" .img-center }

A 16-bit integer is used to encode values between 0 and 65535 (between 0x0000 and 0xffff). This simple modeling makes it possible to distinguish two instantaneous `pressed` and `released` states, as well as a durable `held` state whose value tells us the number of `loop()` cycles during which the button has been held down.

As for the `Led` component, we will also define some methods on the `Button` component to easily determine its state:

```cpp
// ----------------------------------------------------------------------------
// Definition of the Button component
// ----------------------------------------------------------------------------

struct Button {
    // state variables
    uint8_t  pin;
    bool     lastReading;
    uint32_t lastDebounceTime;
    uint16_t state;

    // methods determining the logical state of the button
    bool pressed()                { return state == 1; }
    bool released()               { return state == 0xffff; }
    bool held(uint16_t count = 0) { return state > 1 + count && state < 0xffff; }
};
```

The `Button` component now has methods that can inform us directly about its `pressed`, `released` or `held` state. All these states can be deducted from the integer value of the state variable `state`, according to the diagram above.

Note in particular that the `held()` function accepts an optional `count` argument to specify the number of cycles of the `loop()` function during which the button must be held down for the `true` value to be returned.

We still have to define a `read()` method to be able to read the physical state of the button and determine the value of the `state` variable:

```cpp
// ----------------------------------------------------------------------------
// Definition of the Button component
// ----------------------------------------------------------------------------

struct Button {
    // state variables
    uint8_t  pin;
    bool     lastReading;
    uint32_t lastDebounceTime;
    uint16_t state;

    // methods determining the logical state of the button
    bool pressed()                { return state == 1; }
    bool released()               { return state == 0xffff; }
    bool held(uint16_t count = 0) { return state > 1 + count && state < 0xffff; }

    // method for reading the physical state of the button
    void read() {
        // reads the voltage on the pin connected to the button
        bool reading = digitalRead(pin);

        // if the logic level has changed since the last reading,
        // we reset the timer which counts down the necessary time
        // beyond which we can consider that the bouncing effect
        // has passed.
        if (reading != lastReading) {
            lastDebounceTime = millis();
        }

        // from the moment we're out of the bouncing phase
        // the actual status of the button can be determined
        if (millis() - lastDebounceTime > DEBOUNCE_DELAY) {
            // don't forget that the read pin is pulled-up
            bool pressed = reading == LOW;
            if (pressed) {
                     if (state  < 0xfffe) state++;
                else if (state == 0xfffe) state = 2;
            } else if (state) {
                state = state == 0xffff ? 0 : 0xffff;
            }
        }

        // finally, each new reading is saved
        lastReading = reading;
    }
};
```

Here we go, the `Button` component is ready. Now let's see how to use it.


## How to use the Button component

Let's start by defining a `Button` object:

```cpp
// ----------------------------------------------------------------------------
// Definition of global variables
// ----------------------------------------------------------------------------

Led    led    = { LED_PIN, false };
Button button = { BTN_PIN, HIGH, 0, 0 };
```

Then let's initialize the reading pin of the button:

```cpp
// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

void setup() {
    pinMode(led.pin,    OUTPUT);
    pinMode(button.pin, INPUT);
}
```

It is now very easy to switch the LED on and off with the button. For example, suppose you want to turn the LED on while the button is held down, and off when it is released:

```cpp
// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    button.read();

         if (button.held())     led.on = true;
    else if (button.released()) led.on = false;
    
    led.update();
}
```

Let's compile and upload the program and see what happens:

<div class="video-wrapper with-caption shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo-button-held.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

Super simple, isn't it?
{: .caption }

Now, instead, let's make sure that every time the button is pressed, the LED status is reversed :

```cpp
// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    button.read();

    if (button.pressed()) led.on = !led.on;

    led.update();
}
```

And here's what happens now:

<div class="video-wrapper with-caption shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo-button-pressed.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

It was even simpler!
{: .caption }

You can update the baseline project by going to the `remote-control-with-websocket` directory and executing the following `git` command:

```
git checkout v0.2
```

You can also download the updated `main.cpp` file and replace yours in your own project:

<div class="files">
    <a class="icon" href="{% include github-download.html file="/v0.2/src/main.cpp" %}" download="main.cpp">
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
#define BTN_PIN 22

// ----------------------------------------------------------------------------
// Definition of global constants
// ----------------------------------------------------------------------------

const uint8_t DEBOUNCE_DELAY = 10; // in milliseconds

// ----------------------------------------------------------------------------
// Definition of the LED component
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
// Definition of the Button component
// ----------------------------------------------------------------------------

struct Button {
    // state variables
    uint8_t  pin;
    bool     lastReading;
    uint32_t lastDebounceTime;
    uint16_t state;

    // methods determining the logical state of the button
    bool pressed()                { return state == 1; }
    bool released()               { return state == 0xffff; }
    bool held(uint16_t count = 0) { return state > 1 + count && state < 0xffff; }

    // method for reading the physical state of the button
    void read() {
        // reads the voltage on the pin connected to the button
        bool reading = digitalRead(pin);

        // if the logic level has changed since the last reading,
        // we reset the timer which counts down the necessary time
        // beyond which we can consider that the bouncing effect
        // has passed.
        if (reading != lastReading) {
            lastDebounceTime = millis();
        }

        // from the moment we're out of the bouncing phase
        // the actual status of the button can be determined
        if (millis() - lastDebounceTime > DEBOUNCE_DELAY) {
            // don't forget that the read pin is pulled-up
            bool pressed = reading == LOW;
            if (pressed) {
                     if (state  < 0xfffe) state++;
                else if (state == 0xfffe) state = 2;
            } else if (state) {
                state = state == 0xffff ? 0 : 0xffff;
            }
        }

        // finally, each new reading is saved
        lastReading = reading;
    }
};

// ----------------------------------------------------------------------------
// Definition of global variables
// ----------------------------------------------------------------------------

Led    led    = { LED_PIN, false };
Button button = { BTN_PIN, HIGH, 0, 0 };

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

void setup() {
    pinMode(led.pin,    OUTPUT);
    pinMode(button.pin, INPUT);
}

// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    button.read();

    if (button.pressed()) led.on = !led.on;

    led.update();
}
```


[LED Setup]({{ '/led-setup/' | relative_url }}){: .btn }
[Web UI Design]({{ '/web-ui-design/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[bouncing]: {{ 'images/button-bouncing-effect.png' | relative_url }}
[states]:   {{ 'images/button-state-16-bits-1331x233.png' | relative_url }}

[hackaday]: https://hackaday.com/2015/12/09/embed-with-elliot-debounce-your-noisy-buttons-part-i/
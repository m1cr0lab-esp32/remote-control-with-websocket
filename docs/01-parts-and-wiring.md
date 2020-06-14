---
layout: default
title: Parts and Wiring
nav_order: 2
permalink: /parts-and-wiring/
---

# Circuit Design

## Required parts

- 1 [ESP32][parts-esp32]
- 2 [resistors][parts-res] (1 x 220 Ω and 1 x 10 kΩ)
- 1 [LED][parts-led]
- 1 [push button][parts-btn]
- 2 [breadboards][parts-bb] (ESP32 dev boards are usually too large to fit on a single breadboard)
- [jumper wires][parts-wire1] (male to male) or a *"hook-up"* [wire spool set][parts-wire2] (22 AWG)
- a [wire stripper][parts-strip] (in case you use spools of wire)

## Wiring diagram

You can also get by with just one breadboard, by running the wires under the ESP32 board. But to make the schematic easier to read, I preferred to wire the components on two breadboards.

On my side, I used a 456 holes breadboard which has 6 holes per column and which allows to put all the components on a single breadboard.

![Wiring diagram]({{ 'images/wiring-942x1203.png' | relative_url }}){: width="471" }
{: .img-center }

Here are the control pins used:

- **GPIO 22** is connected to the power line of the push button
- **GPIO 26** is connected to the anode of the LED

Note that I also used a 10 µF electrolytic capacitor between the EN pin and the ground, but it is only there to fix a manufacturing defect of my ESP32 board which does not automatically put itself in flashing mode when I want to upload a new program through the USB port.

![Circuit Overview][circuit]{: width="512" .shadow }
{: .img-center }

This problem is common on boards that are not made by Espressif itself, and I found this trick here:

[\[SOLVED\] Failed to connect to ESP32: Timed out waiting for packet header][timeout]

## Pull-up resistor

You can notice that we’ve fitted the push button with a pull-up resistor. Let’s see why...

The digital pin to which the button is connected (GPIO23) must always be able to read a HIGH or LOW signal. And this signal must be able to be inverted depending on whether or not the button is pressed. Above all, however, care must be taken to ensure that this signal is not floating, otherwise the behaviour of the circuit may become erratic. And this is precisely the role of a pull-up or pull-down resistor.

A pull-up resistor is used to keep the pin in a default HIGH logic state by connecting it to the 3.3V supply line. A pull-down resistor does exactly the opposite and keeps the pin in a default LOW logic state by connecting it to ground. It is important to specify that this is not a specific type of resistor: it is a common resistor. Its place in the circuit gives it this name because of the function it performs.

I used a pull-up resistor here. In this way, the button status reading pin is kept in a default HIGH state and, when the button is pressed, its state will change to LOW.

![Pull-Up Resistor]({{ 'images/pull-up-button.png' | relative_url }}){: width="519" }
{: .img-center }

Why use a 10 kΩ resistor? Does this value matter? Yes and no... in fact we usually use a resistor between 4.7 kΩ and 100 kΩ. The lower you go down to 4.7 kΩ, the more you increase the leakage current, which therefore wastes energy. The closer you get to 100 kΩ, the greater the risk of interfering with the internal circuitry of the microcontroller, causing erratic detection on the reading pin. I suggest going with 10 kΩ because they are common resistor values and people reading your schematic will be more likely to understand that it's just a pull-up resistor.


[Overview]({{ '/' | relative_url }}){: .btn }
[PlatformIO IDE]({{ '/platformio-ide/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[circuit]:     {{ 'images/circuit-1024x768.jpg' | relative_url }}

[parts-esp32]: https://makeradvisor.com/tools/esp32-dev-board-wi-fi-bluetooth/
[parts-res]:   https://makeradvisor.com/tools/resistors-kits/
[parts-led]:   https://makeradvisor.com/tools/3mm-5mm-leds-kit-storage-box/
[parts-btn]:   https://makeradvisor.com/tools/pushbuttons-kit/
[parts-bb]:    https://makeradvisor.com/tools/mb-102-solderless-breadboard-830-points/
[parts-wire1]: https://makeradvisor.com/tools/jumper-wires-kit-120-pieces/
[parts-wire2]: https://thepihut.com/products/prototyping-wire-spool-set
[parts-strip]: https://makeradvisor.com/best-self-adjusting-wire-stripper/
[timeout]:     https://randomnerdtutorials.com/solved-failed-to-connect-to-esp32-timed-out-waiting-for-packet-header/
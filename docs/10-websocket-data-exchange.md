---
layout: default
title: WebSocket Data Exchange
nav_order: 11
permalink: /websocket-data-exchange/
---

# WebSocket Data Exchange

We will start by setting up a notification feature that will notify all WebSocket clients when an operator presses the button to turn the wired LED on or off. Keep in mind that it is possible to interact with multiple clients connected to the same WebSocket server.


## Sending data from the server

The implementation at the server level is very simple:

```cpp
// ----------------------------------------------------------------------------
// Sending data to WebSocket clients
// ----------------------------------------------------------------------------

void notifyClients() {
    ws.textAll(led.on ? "on" : "off");
}

// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    ws.cleanupClients();

    button.read();

    if (button.pressed()) { //
        led.on = !led.on;   //
        notifyClients();    // <-- change happens here
    }
    
    onboard_led.on = millis() % 1000 < 50;

    led.update();
    onboard_led.update();
}
```

The `AsyncWebSocket` class provides a `textAll()` method for sending the same message to all clients at the same time. Convenient, isn't it?

Now let's look at how to receive these notifications at the client level. To do this, edit the `index.js` script and add the following:

```js
// [ ... ]

function initWebSocket() {
    console.log('Trying to open a WebSocket connection...');
    websocket = new WebSocket(gateway);
    websocket.onopen    = onOpen;
    websocket.onclose   = onClose;
    websocket.onmessage = onMessage; // <-- add this line
}

// [ ... ]

function onMessage(event) {
    console.log(`Received a notification from ${event.origin}`);
    console.log(event);
}
```

Upload the `data` folder on the SPIFFS again, then reload the web interface in the browser. If not already done, open the browser's Javascript console, then press the wired push button on the breadboard twice to turn the LED on and off. You should see this in the Javascript console:

```
Trying to open a WebSocket connection...
Connection opened
Received a notification from ws://192.168.1.32
MessageEvent {isTrusted: true, data: "on", origin: "ws://192.168.1.32", lastEventId: "", source: null, …}
Received a notification from ws://192.168.1.32
MessageEvent {isTrusted: true, data: "off", origin: "ws://192.168.1.32", lastEventId: "", source: null, …}
```

You can see that the client receives a `MessageEvent` type event that has a `data` property which is equal to the string sent by the ESP32 each time the operator pressed the button.

It is therefore very easy to interpret this data to dynamically modify the web interface and turn on or off the virtual LED by modifying the class of the corresponding CSS element:

```js
function onMessage(event) {
    document.getElementById('led').className = event.data;
}
```

> Refer to the end of the [Web UI Design][webui] chapter if you have a memory lapse on this subject...

Upload the `data` folder on the SPIFFS again and reload the web interface in the browser. Now, each time you press the push button to turn the LED on or off, its virtual version also changes its status on the web interface.

Open a second browser window by pointing to the URL of the web interface exposed by the ESP32. Then a third. Even launch another browser. Do the same thing from your smartphone. Use your partner's as well... What do you notice every time you press the push button? All open web interfaces react immediately to the physical event.

<div class="video-wrapper full-width shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo-ws-pushbutton.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

> You must realize here that **the notification is spontaneously sent by the WebSocket server**! At no time does the client query the server for the new status of the LED. This feature is indeed possible with the WebSocket protocol, whereas it is not possible when using the `XMLHttpRequest` object, as in the [Random Nerd Tutorials][rnt].

Feel free to press the button several times quickly. You will see that all clients will update immediately. This is a strength of WebSockets, which are much faster and lighter than the HTTP protocol.


## Sending data from the client

Now let's see how to deal with the reciprocal by turning the physical LED on or off when the user presses the **Toggle** button on the web interface.

Edit the `index.js` file and add the following:

```js
// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

window.addEventListener('load', onLoad);

function onLoad(event) {
    initWebSocket();
    initButton(); // <-- add this line
}

// ----------------------------------------------------------------------------
// Button handling
// ----------------------------------------------------------------------------

function initButton() {
    document.getElementById('toggle').addEventListener('click', onToggle);
}

function onToggle(event) {
    websocket.send('toggle');
}
```

Then upload the new changes to SPIFFS.

We must now make sure to take into account the `toogle_led` message sent by the client(s) on the server side. Things are a bit heavier to deal with in C++. We must indeed take some precautions to analyze the received message. Indeed, the data is received in a buffer that can be split into several parts if the amount of data is too large.

In order not to complicate things here, we will limit ourselves to processing relatively light data (the message to be processed here is very short). But you can refer to the documentation of the [ESPAsyncWebServer][espasync] library if you want to go further and discover how to process larger quantities of data.

Here's how to proceed:

```cpp
// ----------------------------------------------------------------------------
// WebSocket initialization
// ----------------------------------------------------------------------------

void notifyClients() {
    ws.textAll(led.on ? "on" : "off");
}

void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {
        data[len] = 0;
        if (strcmp((char*)data, "toggle") == 0) {
            led.on = !led.on;
            notifyClients();
        }
    }
}

void onEvent(AsyncWebSocket       *server,
             AsyncWebSocketClient *client,
             AwsEventType          type,
             void                 *arg,
             uint8_t              *data,
             size_t                len) {

    switch (type) {
        case WS_EVT_CONNECT:
            Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
            break;
        case WS_EVT_DISCONNECT:
            Serial.printf("WebSocket client #%u disconnected\n", client->id());
            break;
        case WS_EVT_DATA:
            handleWebSocketMessage(arg, data, len);
            break;
        case WS_EVT_PONG:
        case WS_EVT_ERROR:
            break;
    }
}

void initWebSocket() {
    ws.onEvent(onEvent);
    server.addHandler(&ws);
}
```

This time we are interested in the event of type `WS_EVT_DATA`. The `handleWebSocketMessage()` function is defined to process this event. This function takes three arguments to decode the received data. The first step is to ensure that the data are received in full and in one piece, and that they correspond to textual data (binary data can also be processed).

If all the conditions are met, we convert the sequence of bytes received into a string, which we compare with the expected message. If the received message is indeed `toggle`, then we invert the state of the `led` object. This will turn the physical LED on or off at the end of the `loop()` function. But don't forget to send a status change notification to all connected clients so that they can adjust the state of the virtual LED on their respective web interface. So don't forget to move the `notifyClients()` function so that it is defined before it is used.

There you go! You can recompile the code and upload it to the ESP32, then reload the web interface in the browser of your PC and/or smartphone. This time, everything works, whether you press the physical or virtual button:

<div class="video-wrapper full-width with-caption shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

Magic, right?
{: .caption }

We have only dealt here with the sending of messages in the form of simple strings. But it can be very useful to structure the data exchanged between the server and the clients in a more structured way. We will look at this aspect in the next chapter using the JSON format.

In the meantime, you can now update the baseline project by going to the `remote-control-with-websocket` directory and executing the following `git` command:

```
git checkout v0.8
```

You can also download the updated `main.cpp` and `index.js` files and replace yours in your own project:

<div class="files">
    <a class="icon" href="{% include github-download.html file="/v0.8/src/main.cpp" %}" download="main.cpp">
        <img width="64" src="{{ 'images/file-cpp-icon-178x192.png' | relative_url }}">
        <p class="filename">main.cpp</p>
    </a>
    <a class="icon" href="{% include github-download.html file="/v0.8/data/index.js" %}" download="index.js">
        <img width="64" src="{{ 'images/file-js-icon-178x192.png' | relative_url }}">
        <p class="filename">index.js</p>
    </a>
</div>

Here is the complete code for this chapter:


### Server side

```cpp
/**
 * ----------------------------------------------------------------------------
 * ESP32 Remote Control with WebSocket
 * ----------------------------------------------------------------------------
 * © 2020 Stéphane Calderoni
 * ----------------------------------------------------------------------------
 */

#include <Arduino.h>
#include <SPIFFS.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// ----------------------------------------------------------------------------
// Definition of macros
// ----------------------------------------------------------------------------

#define LED_PIN   26
#define BTN_PIN   22
#define HTTP_PORT 80

// ----------------------------------------------------------------------------
// Definition of global constants
// ----------------------------------------------------------------------------

// Button debouncing
const uint8_t DEBOUNCE_DELAY = 10; // in milliseconds

// WiFi credentials
const char *WIFI_SSID = "YOUR_WIFI_SSID";
const char *WIFI_PASS = "YOUR_WIFI_PASSWORD";

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

Led    onboard_led = { LED_BUILTIN, false };
Led    led         = { LED_PIN, false };
Button button      = { BTN_PIN, HIGH, 0, 0 };

AsyncWebServer server(HTTP_PORT);
AsyncWebSocket ws("/ws");

// ----------------------------------------------------------------------------
// SPIFFS initialization
// ----------------------------------------------------------------------------

void initSPIFFS() {
  if (!SPIFFS.begin()) {
    Serial.println("Cannot mount SPIFFS volume...");
    while (1) {
        onboard_led.on = millis() % 200 < 50;
        onboard_led.update();
    }
  }
}

// ----------------------------------------------------------------------------
// Connecting to the WiFi network
// ----------------------------------------------------------------------------

void initWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.printf("Trying to connect [%s] ", WiFi.macAddress().c_str());
  while (WiFi.status() != WL_CONNECTED) {
      Serial.print(".");
      delay(500);
  }
  Serial.printf(" %s\n", WiFi.localIP().toString().c_str());
}

// ----------------------------------------------------------------------------
// Web server initialization
// ----------------------------------------------------------------------------

String processor(const String &var) {
    return String(var == "STATE" && led.on ? "on" : "off");
}

void onRootRequest(AsyncWebServerRequest *request) {
  request->send(SPIFFS, "/index.html", "text/html", false, processor);
}

void initWebServer() {
    server.on("/", onRootRequest);
    server.serveStatic("/", SPIFFS, "/");
    server.begin();
}

// ----------------------------------------------------------------------------
// WebSocket initialization
// ----------------------------------------------------------------------------

void notifyClients() {
    ws.textAll(led.on ? "on" : "off");
}

void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {
        data[len] = 0;
        if (strcmp((char*)data, "toggle") == 0) {
            led.on = !led.on;
            notifyClients();
        }
    }
}

void onEvent(AsyncWebSocket       *server,
             AsyncWebSocketClient *client,
             AwsEventType          type,
             void                 *arg,
             uint8_t              *data,
             size_t                len) {

    switch (type) {
        case WS_EVT_CONNECT:
            Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
            break;
        case WS_EVT_DISCONNECT:
            Serial.printf("WebSocket client #%u disconnected\n", client->id());
            break;
        case WS_EVT_DATA:
            handleWebSocketMessage(arg, data, len);
            break;
        case WS_EVT_PONG:
        case WS_EVT_ERROR:
            break;
    }
}

void initWebSocket() {
    ws.onEvent(onEvent);
    server.addHandler(&ws);
}

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

void setup() {
    pinMode(onboard_led.pin, OUTPUT);
    pinMode(led.pin,         OUTPUT);
    pinMode(button.pin,      INPUT);

    Serial.begin(115200); delay(500);

    initSPIFFS();
    initWiFi();
    initWebSocket();
    initWebServer();
}

// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    ws.cleanupClients();

    button.read();

    if (button.pressed()) {
        led.on = !led.on;
        notifyClients();
    }
    
    onboard_led.on = millis() % 1000 < 50;

    led.update();
    onboard_led.update();
}
```


### Client side

```js
/**
 * ----------------------------------------------------------------------------
 * ESP32 Remote Control with WebSocket
 * ----------------------------------------------------------------------------
 * © 2020 Stéphane Calderoni
 * ----------------------------------------------------------------------------
 */

var gateway = `ws://${window.location.hostname}/ws`;
var websocket;

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

window.addEventListener('load', onLoad);

function onLoad(event) {
    initWebSocket();
    initButton();
}

// ----------------------------------------------------------------------------
// WebSocket handling
// ----------------------------------------------------------------------------

function initWebSocket() {
    console.log('Trying to open a WebSocket connection...');
    websocket = new WebSocket(gateway);
    websocket.onopen    = onOpen;
    websocket.onclose   = onClose;
    websocket.onmessage = onMessage;
}

function onOpen(event) {
    console.log('Connection opened');
}

function onClose(event) {
    console.log('Connection closed');
    setTimeout(initWebSocket, 2000);
}

function onMessage(event) {
    document.getElementById('led').className = event.data;
}

// ----------------------------------------------------------------------------
// Button handling
// ----------------------------------------------------------------------------

function initButton() {
    document.getElementById('toggle').addEventListener('click', onToggle);
}

function onToggle(event) {
    websocket.send('toggle');
}
```


[WebSocket Setup]({{ '/websocket-setup/' | relative_url }}){: .btn }
[WebSocket & JSON]({{ '/websocket-and-json/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[espasync]: https://me-no-dev.github.io/ESPAsyncWebServer/
[rnt]:      https://randomnerdtutorials.com/
[webui]:    {{ '/web-ui-design/' | relative_url }}
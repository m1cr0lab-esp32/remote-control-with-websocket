---
layout: default
title: WebSocket Setup
nav_order: 10
permalink: /websocket-setup/
---

# Setting Up WebSocket

WebSocket is a computer communications protocol, providing full-duplex communication channels over a single TCP connection. The WebSocket protocol enables interaction between a web browser (or other client application) and a web server with lower overhead than half-duplex alternatives such as HTTP polling, facilitating real-time data transfer from and to the server. This is made possible by providing a standardized way for the server to send content to the client without being first requested by the client, and allowing messages to be passed back and forth while keeping the connection open. In this way, a two-way ongoing conversation can take place between the client and the server.

The WebSocket protocol specification defines `ws` (WebSocket) and `wss` (WebSocket Secure) as two new uniform resource identifier (URI) schemes that are used for unencrypted and encrypted connections, respectively. We'll settle for unencrypted connections here.


## Initializing the WebSocket server

The [ESPAsyncWebServer][espasync] library includes a WebSocket plugin which lets you define different WebSocket locations to connect to without starting another listening service or using different port. It is therefore very easy to implement this protocol on the basis of what we have already achieved. Let's define a new global variable `ws` to handle the WebSocket server, attached to the `/ws` path:

```cpp
// ----------------------------------------------------------------------------
// Definition of global variables
// ----------------------------------------------------------------------------

Led    onboard_led = { LED_BUILTIN, false };
Led    led         = { LED_PIN, false };
Button button      = { BTN_PIN, HIGH, 0, 0 };

AsyncWebServer server(HTTP_PORT);
AsyncWebSocket ws("/ws");
```

The WebSocket server initialization procedure follows easily:

```cpp
// ----------------------------------------------------------------------------
// WebSocket initialization
// ----------------------------------------------------------------------------

void initWebSocket() {
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
    initWebSocket(); // <-- add this line
    initWebServer();
}
```


## Configuring the WebSocket server

We now need to define an event listener to handle the different asynchronous steps of the WebSocket protocol. This event handler can be implemented by defining the `onEvent()` function, which must comply with the [AwsEventHandler][handler] interface:

```cpp
// ----------------------------------------------------------------------------
// WebSocket initialization
// ----------------------------------------------------------------------------

void onEvent(AsyncWebSocket       *server,  //
             AsyncWebSocketClient *client,  //
             AwsEventType          type,    // the signature of this function is defined
             void                 *arg,     // by the `AwsEventHandler` interface
             uint8_t              *data,    //
             size_t                len) {   //

    // we are going to add here the handling of
    // the different events defined by the protocol
}

void initWebSocket() {
    ws.onEvent(onEvent);
    server.addHandler(&ws);
}
```

The `type` argument characterizes the nature of the event that occurs. It can take the following values:

- `WS_EVT_CONNECT` when a client has logged in,
- `WS_EVT_DISCONNECT` when a client has logged out,
- `WS_EVT_DATA` when a data packet is received from the client.
- `WS_EVT_PONG` in response to a *ping* request,
- `WS_EVT_ERROR` when an error is received from the client,

In this tutorial, we'll look at the first three.


## Handling client connections

Let's start by looking at how to handle the connection between the server and the clients.

On the server side, things are very simple:

```cpp
// ----------------------------------------------------------------------------
// WebSocket initialization
// ----------------------------------------------------------------------------

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
        case WS_EVT_PONG:
        case WS_EVT_ERROR:
            break;
    }
}
```

On the client side, we'll add some code in the `index.js` script, which is hosted on the SPIFFS, and which has been empty until now:

```js
var gateway = `ws://${window.location.hostname}/ws`;
var websocket;

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

window.addEventListener('load', onLoad);

function onLoad(event) {
    initWebSocket();
}

// ----------------------------------------------------------------------------
// WebSocket handling
// ----------------------------------------------------------------------------

function initWebSocket() {
    websocket = new WebSocket(gateway);
}
```

The `gateway` variable represents the entry point to our WebSocket interface. It is dynamically resolved using `window.location.hostname` and automatically translated by the following URI:

```
ws://192.168.1.32/ws
```

The connection is initiated as soon as the web interface is fully loaded in the browser with the following instruction:

```js
websocket = new WebSocket(gateway);
```

That's it, that's all we have to do at this point.

Since we have modified the `index.js` script, we have to upload it to the SPIFFS again:

![Upload File System image][upload]{: width="512" .rounded-5 .shadow }
{: .img-center }

You can now compile the C++ code and upload it to the ESP32. Then point your browser to the URL of the web server:

```
http://192.168.1.32/
```
{: .with-caption }

Replace the IP address with that of your microcontroller
{: .caption }

You should see the following message on your serial monitor:

```
Trying to connect [30:AE:A4:97:F3:0C] ... 192.168.1.32
WebSocket client #1 connected from 192.168.1.100
```

Now, close the web page you just loaded, and you should see the following new message:

```
WebSocket client #1 disconnected
```

Reconnect your browser to ESP32, you should now see:

```
WebSocket client #2 connected from 192.168.1.100
```

Reload the page and you'll see:

```
WebSocket client #2 disconnected
WebSocket client #3 connected from 192.168.1.100
```

You can see that the identifier `#n` obtained from the `client->id()` call is not attached to the IP address from which the connection originates. It is simply a connection identifier that is incremented with each new connection... even if it comes from the same browser. If you reload the page, you close the WebSocket connection, then a new connection is automatically opened as soon as the page is loaded again.

The WebSocket protocol tries to keep the connection active for as long as possible: each client regularly sends a `ping` request to the server, to which the server replies with a `pong`. This is an event that can be captured when the `type` argument of the `onEvent()` function has the value `WS_EVT_PONG`.

Nevertheless, it may happen that the connection is lost for various reasons, or closed intentionally. Browsers sometimes do not correctly close the WebSocket connection, even when the `close()` function is called in Javascript. This will eventually exhaust the web server's resources and will cause the server to crash. To avoid this, the `AsyncWebSocket` class provides an efficient cleaning method: `cleanupClients()`. Periodically calling the `cleanClients()` function from the main `loop()` limits the number of clients by closing the oldest client when the maximum number of clients has been exceeded. This can called be every cycle, however, if you wish to use less power, then calling as infrequently as once per second is sufficient.

Let's make it simple:

```cpp
// ----------------------------------------------------------------------------
// Main control loop
// ----------------------------------------------------------------------------

void loop() {
    ws.cleanupClients(); // <-- add this line

    button.read();

    if (button.pressed()) led.on = !led.on;
    
    onboard_led.on = millis() % 1000 < 50;

    led.update();
    onboard_led.update();
}
```

It is also possible to set up an automatic reconnection mechanism at the client level. Replace the `index.js` script by this one :

```js
var gateway = `ws://${window.location.hostname}/ws`;
var websocket;

// ----------------------------------------------------------------------------
// Initialization
// ----------------------------------------------------------------------------

window.addEventListener('load', onLoad);

function onLoad(event) {
    initWebSocket();
}

// ----------------------------------------------------------------------------
// WebSocket handling
// ----------------------------------------------------------------------------

function initWebSocket() {
    console.log('Trying to open a WebSocket connection...');
    websocket = new WebSocket(gateway);
    websocket.onopen  = onOpen;
    websocket.onclose = onClose;
}

function onOpen(event) {
    console.log('Connection opened');
}

function onClose(event) {
    console.log('Connection closed');
    setTimeout(initWebSocket, 2000);
}
```

Then upload the contents of the `data` folder to the SPIFFS again. Reboot the ESP32 and open the Javascript console of your browser. Now reload the web interface, you should see this in the console:

```
Trying to open a WebSocket connection...
Connection opened
```

Now reboot the ESP32. This will cause the browser to lose its WebSocket connection, and after a few seconds you should see this appear in the browser console:

```
Connection closed
Trying to open a WebSocket connection...
Connection opened
```

As soon as the browser has detected that it has lost the connection to the WebSocket server, it now reopens a new connection.

That's it, WebSocket connection handling is now ready. In the next chapter we will be able to discover how to make the server and the client(s) exchange data.

In the meantime, you can now update the baseline project by going to the `remote-control-with-websocket` directory and executing the following `git` command:

```
git checkout v0.7
```

You can also download the updated `main.cpp` and `index.js` files and replace yours in your own project:

<div class="files">
    <a class="icon" href="{% include github-download.html file="/v0.7/src/main.cpp" %}" download="main.cpp">
        <img width="64" src="{{ 'images/file-cpp-icon-178x192.png' | relative_url }}">
        <p class="filename">main.cpp</p>
    </a>
    <a class="icon" href="{% include github-download.html file="/v0.7/data/index.js" %}" download="index.js">
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

    if (button.pressed()) led.on = !led.on;
    
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
}

// ----------------------------------------------------------------------------
// WebSocket handling
// ----------------------------------------------------------------------------

function initWebSocket() {
    console.log('Trying to open a WebSocket connection...');
    websocket = new WebSocket(gateway);
    websocket.onopen  = onOpen;
    websocket.onclose = onClose;
}

function onOpen(event) {
    console.log('Connection opened');
}

function onClose(event) {
    console.log('Connection closed');
    setTimeout(initWebSocket, 2000);
}
```


[Web Server Setup]({{ '/web-server-setup/' | relative_url }}){: .btn }
[WebSocket Data Exchange]({{ '/websocket-data-exchange/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[upload]:   {{ 'images/demo-spiffs-upload-1200x1022.gif' | relative_url }}

[espasync]: https://me-no-dev.github.io/ESPAsyncWebServer/
[handler]:  https://github.com/me-no-dev/ESPAsyncWebServer/blob/f13685ee97675be2ac9502d177d3024ebc49c1e0/src/AsyncWebSocket.h#L240
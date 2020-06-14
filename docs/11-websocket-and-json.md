---
layout: default
title: WebSocket & JSON
nav_order: 12
permalink: /websocket-and-json/
---

# WebSocket Data Exchange with JSON

So far, we have only exchanged simple strings between the server and its clients using the WebSocket protocol. As long as the information transmitted remains simple to analyze, we can be satisfied with it. But sometimes we may need to exchange more complex data. In this case, it is preferable to structure the data so that it can be analysed more easily. This is where the JSON format comes in. JSON is a way to represent complex data structures as text. The resulting string is called a JSON document and can then be sent via the network or saved to a file.

For example, we may need to communicate more accurate information when the server needs to notify its clients with the LED status change:

```json
{
    "device": "led",
    "status": "on",
    "time":   1591510722
}
```

The text above describes the *prettified* representation in JSON format of an object composed of:

- a string named `device` with the value `led`,
- another string named `status` with the value `on`,
- an integer named `time` with the value `1591510722`.

JSON ignores spaces and line breaks, so the same object can be represented with the following *minified* JSON document:

```json
{"device":"led","status":"on","time":1591510722}
```

<!-- Avant de pouvoir transmettre cet objet à ses clients, le serveur devra convertir cette donnée structurée en une suite d'octets. C'est ce qu'on appelle la serialisation. -->

Before the server can transmit this object to its clients, it will have to convert this structured data into a sequence of bytes. This is called **serialization**. In the context of JSON, serialization is the creation of a JSON document from an object in memory, and **deserialization** is the reverse operation.

Here the serialization remains quite simple and can be done easily with the `sprintf()` function:

```cpp
sprintf(
    buffer,
    "{\"device\":\"%s\",\"status\":\"%s\",\"time\":\"%u\"}", // format
    "led",                                                   // device
    led.on ? "on" : "off",                                   // status
    millis()                                                 // time
);
```

You must of course provide a `buffer` large enough to hold the entire serialized data.

JSON makes it possible to structure much more complex data than that. It allows you to manage arrays of objects. Each object can encapsulate other objects in turn, or even arrays, etc.

As a result, serialization and deserialization processes can be laborious to implement. But, fortunately, there are plenty of libraries that can handle this for us. In this tutorial, we'll use the **excellent** [ArduinoJson][arduinojson] library developed by Benoît Blanchon.

> **Important**  
> The following explanations are only compatible with version **6** of the **ArduinoJson** library, which is the most recent version at the time of writing this tutorial.

My intention here is not to give you a complete course on the JSON format and its implementation with the **ArduinoJson** library. For that, please refer to its [documentation][arduinojsondoc], which is very complete. Nevertheless, I will present some basics to introduce you to its use.


## Client notification

### On the server side

When the LED changes state, the server transmits the new state to all clients. And so far, we've been doing it like this:

```cpp
void notifyClients() {
    ws.textAll(led.on ? "on" : "off");
}
```

As long as the data to be transmitted is simple, one can simply proceed as follows, without the need to use a library specialized in the processing of the JSON format:

```cpp
void notifyClients() {
    char buffer[17];
    sprintf(buffer, "{\"status\":\"%s\"}", led.on ? "on" : "off");
    ws.textAll(buffer);
}
```

Note that to store the string encoded in JSON format, a sufficiently large `buffer` is required. Indeed, depending on the state of the LED, `on` or `off`, the longest string to be encoded is the following:

```json
{"status":"off"}
```

This string is exactly 16 characters long, to which it is necessary to think of adding the termination character `\0` of the strings in C. Our buffer must therefore be able to hold a maximum of **17** characters. This is the reason for the following statement:

```cpp
char buffer[17];
```

But let's look instead at how to implement the same thing using the **ArduinoJson** library. First of all, we need to add the library to our project. To do this, we just need to edit our `platformio.ini` file and add a new dependency:

```ini
[env:esp32doit-devkit-v1]
platform      = espressif32
board         = esp32doit-devkit-v1
framework     = arduino
upload_speed  = 921600
monitor_speed = 115200
lib_deps      = ESP Async WebServer, ArduinoJson
```

Then we need to include the library header file at the beginning of our program:

```cpp
#include <Arduino.h>
#include <SPIFFS.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <ArduinoJson.h>
```

One of the fundamental characteristics of the **ArduinoJson** library is its fixed memory allocation strategy. You first have to create a `JsonDocument` to reserve a specified amount of memory. The memory of the `JsonDocument` can be either in the **stack** or in the **heap**. The location depends on the derived class you choose. If you use a `StaticJsonDocument`, it will be in the stack. If you use a `DynamicJsonDocument`, it will be in the heap.

Since our `JsonDocument` is small, we can keep it in the stack. By using the stack, we reduce the size of the executable and improve the performance, because we avoid the overhead due to the management of the heap. We don't need to engage a dynamic allocation strategy here, and we will avoid the problem of memory fragmentation at the same time.

When you create a `JsonDocument`, you must specify its capacity in bytes. In the case of a `StaticJsonDocument`, you set the capacity via a template parameter:

```cpp
StaticJsonDocument<size> json;
```

As `size` is a template parameter, you cannot use a variable. Instead, you must use a constant expression, which means that the value must be computed at compile-time. The compiler manages the stack, so it needs to know the size of each variable when it compiles the program.

How to determine the capacity? To answer this question, you need to know what **ArduinoJson** stores in the `JsonDocument`. **ArduinoJson** needs to store a data structure that mirrors the hierarchy of objects in the JSON document. In other words, the `JsonDocument` contains objects which relate to one another the same way they do in the JSON document.

Therefore, the capacity of the `JsonDocument` highly depends on the complexity of the JSON document. If it’s just one object with few members, like our example, a few dozens of bytes are enough. If it’s a massive JSON document, like OpenWeatherMap’s response, up to a hundred kilobytes are needed.

**ArduinoJson** provides macros for computing precisely the capacity of the `JsonDocument`. The macro to compute the size of an object is `JSON_OBJECT_SIZE()`. It takes one argument: the number of members in the object. Here is how to compute the capacity for our sample document:

```cpp
const uint8_t size = JSON_OBJECT_SIZE(1);
StaticJsonDocument<size> json;
```

The `JsonDocument` is currently empty. An empty `JsonDocument` automatically becomes an object when we add members to it. We do that with the subscript operator `[]`:

```cpp
const uint8_t size = JSON_OBJECT_SIZE(1);
StaticJsonDocument<size> json;
json["status"] = led.on ? "on" : "off";
```

The memory usage is now `JSON_OBJECT_SIZE(1)`, so the `JsonDocument` is full. When the `JsonDocument` is full, you cannot add more members, so don’t forget to increase the capacity if you need.

Now, it’s time to serialize it into a JSON document. There are several ways to do that. Here we will look at how to write the JSON document to memory. We could use a `String`, but as you know, I prefer avoiding dynamic memory allocation. Instead, we’d use a good old `char[]` with enough space to receive the longest formatted string we need to encode:

```cpp
char data[17];
```

Then, to produce a JSON document from a `JsonDocument`, we simply need to call `serializeJson()`:

```cpp
char data[17];
size_t len = serializeJson(json, data);
```

The `serializeJson()` function returns the effective length of the serialized JSON document. The value of this `len` variable will be useful for sending the JSON document to WebSocket clients. Here is the complete definition of the rewritten `notifyClients()` function:

```cpp
void notifyClients() {
    const uint8_t size = JSON_OBJECT_SIZE(1);
    StaticJsonDocument<size> json;
    json["status"] = led.on ? "on" : "off";

    char data[17];
    size_t len = serializeJson(json, data);
    ws.textAll(data, len);
}
```

Using the **ArduinoJson** library on this very simple case probably seems overkill, and you're right. But I wanted to make you aware of its use, which will prove extremely life-saving in the case of more complex data. I encourage you to carefully read the [documentation][arduinojsondoc] of this library, which details many useful use cases.


### On the client side

Now that the server broadcasts its notifications in JSON format, we need to take this into account on the client side and rewrite the `onMessage()` function that receives these notifications in the `index.js` file:

```js
function onMessage(event) {
    let data = JSON.parse(event.data);
    document.getElementById('led').className = data.status;
}
```

You can see that this new data processing is extremely simple. We start by decoding the received object in JSON format to make it a Javascript object. And then we can easily read the value of its `status` member. It's as simple as that!


## Server notification

This time, when the user presses the **Toggle** button on the web interface, we will send a notification to the server in the form of a message in JSON format.

### On the client side

We'll slightly modify the `onToggle()` function in the `index.js` file to send the following JSON message:

```json
{"action":"toggle"}
```

It's quite simple:

```js
function onToggle(event) {
    websocket.send(JSON.stringify({'action':'toggle'}));
}
```

## On the server side

To make things a little easier, we assume here that the structure of the expected JSON message is known in advance. The JSON document consists of a JSON object with a single member named `action`, whose value is a string.

We will rewrite the `handleWebSocketMessage()` function to handle messages received in JSON format:

```cpp
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        // We'll see in the following how to decode the received message
        
    }
}
```

Since our `JsonDocument` is small, we can keep it in the stack:

```cpp
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        const uint8_t size = JSON_OBJECT_SIZE(1);
        StaticJsonDocument<size> json;
        
    }
}
```

Now that the `JsonDocument` is ready, we can parse the input data with the `deserializeJson()` function:

```cpp
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        const uint8_t size = JSON_OBJECT_SIZE(1);
        StaticJsonDocument<size> json;
        DeserializationError err = deserializeJson(json, data);
        
    }
}
```

The `deserializeJson()` function returns a `DeserializationError` that tells whether the operation was successful. `DeserializationError` converts implicitly to bool, so we don't need to identify the nature of the error by searching for the error code and comparing it to the predefined constants of the `DeserializationError` class. We can simply write:

```cpp
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        const uint8_t size = JSON_OBJECT_SIZE(1);
        StaticJsonDocument<size> json;
        DeserializationError err = deserializeJson(json, data);
        if (err) {
            Serial.print(F("deserializeJson() failed with code "));
            Serial.println(err.c_str());
            return;
        }
        
    }
}
```

If no error occurs at the time of deserialization, we can extract the value of the `action` member and compare it to the expected value, i.e. `toggle`:

```cpp
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        const uint8_t size = JSON_OBJECT_SIZE(1);
        StaticJsonDocument<size> json;
        DeserializationError err = deserializeJson(json, data);
        if (err) {
            Serial.print(F("deserializeJson() failed with code "));
            Serial.println(err.c_str());
            return;
        }

        const char *action = json["action"];
        if (strcmp(action, "toggle") == 0) {
            led.on = !led.on;
            notifyClients();
        }
        
    }
}
```

If successful, the status of the LED is reversed and all connected clients are notified.

There you go! This time we've come full circle.  
You can upload the contents of the `data` folder to the SPIFFS to update the `index.js` file:

![Upload File System image][upload]{: width="512" .rounded-5 .shadow }
{: .img-center }

Then compile and upload the C++ program to the ESP32. Reload the web user interface in the browser and you will see that everything works exactly as in the previous chapter, except that now the server and clients communicate with messages in JSON format.

<div class="video-wrapper full-width shadow">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

You can now update the baseline project by going to the `remote-control-with-websocket` directory and executing the following `git` command:

```
git checkout v1.0
```

You can also download the updated `main.cpp` and `index.js` files and replace yours in your own project:

<div class="files">
    <a class="icon" href="{% include github-download.html file="/v1.0/src/main.cpp" %}" download="main.cpp">
        <img width="64" src="{{ 'images/file-cpp-icon-178x192.png' | relative_url }}">
        <p class="filename">main.cpp</p>
    </a>
    <a class="icon" href="{% include github-download.html file="/v1.0/data/index.js" %}" download="index.js">
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
#include <ArduinoJson.h>

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
    const uint8_t size = JSON_OBJECT_SIZE(1);
    StaticJsonDocument<size> json;
    json["status"] = led.on ? "on" : "off";

    char data[17];
    size_t len = serializeJson(json, data);
    ws.textAll(data, len);
}

void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
    AwsFrameInfo *info = (AwsFrameInfo*)arg;
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {

        const uint8_t size = JSON_OBJECT_SIZE(1);
        StaticJsonDocument<size> json;
        DeserializationError err = deserializeJson(json, data);
        if (err) {
            Serial.print(F("deserializeJson() failed with code "));
            Serial.println(err.c_str());
            return;
        }

        const char *action = json["action"];
        if (strcmp(action, "toggle") == 0) {
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
    let data = JSON.parse(event.data);
    document.getElementById('led').className = data.status;
}

// ----------------------------------------------------------------------------
// Button handling
// ----------------------------------------------------------------------------

function initButton() {
    document.getElementById('toggle').addEventListener('click', onToggle);
}

function onToggle(event) {
    websocket.send(JSON.stringify({'action':'toggle'}));
}
```


[WebSocket Data Exchange]({{ '/websocket-data-exchange/' | relative_url }}){: .btn }
[Time for Discussion]({{ '/time-for-discussion/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[upload]:         {{ 'images/demo-spiffs-upload-1200x1022.gif' | relative_url }}

[arduinojson]:    https://arduinojson.org/
[arduinojsondoc]: https://arduinojson.org/v6/doc/
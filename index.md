---
layout: default
title: Overview
nav_order: 1
xdate: 2020-04-25
permalink: /
---

I've been following Rui and Sara's work on [Random Nerd Tutorials][rnt] for a few months now, and when it comes to remotely interacting with ESP32 using a web application, I notice the systematic use of the `XMLHttpRequest` object of Javascript. Well, in 2020, we must recognize that there are more modern techniques! One of the most efficient of them is to use the **WebSocket** protocol. And this is what I am going to present to you today.
{: .fs-5 .fw-300 }

- - -

# What are We Going to Do?

The objective here is to focus on the use of WebSocket. To do so, we will wire a fairly simple circuit to illustrate the bidirectional nature of WebSockets. The ESP32 will allow to control the state of a simple LED by reacting both to the physical intervention of a user acting directly on the circuit with a push button, but also remotely with a similar virtual device implemented in a web application. And we will see how to synchronize almost instantaneously the state of physical and virtual devices, whether the user intervenes directly on the circuit, or remotely through the web application.

<div class="img-center" style="display:grid;grid-template-columns:2fr 1fr;grid-gap:20px;align-items:center;">
<img style="width:100%;" class="shadow" src="{{ 'images/circuit-1024x768.jpg' | relative_url }}" alt="Circuit Overview" />
<img style="width:100%;" src="{{ 'images/webapp-off-414x833.png' | relative_url }}" alt="Web Application" />
</div>

Whether it's turning an LED on and off remotely, or controlling a relay to turn on the heating or cooling of your home, or activating your alarm when you leave, or activating the garage door opener when you come home, the process is exactly the same.

Let's see how it works:

<div class="video-wrapper full-width shadow with-caption">
    <video class="video" autoplay muted loop>
        <source src="{{ 'videos/demo.mp4' | relative_url }}" type="video/mp4" />
        Your browser does not support the video tag.
    </video>
</div>

Demo of the application
{: .caption }

Notice the immediacy of the reaction on both smartphones. The web interface is updated at the precise moment the user acts on the physical or virtual device. You won't get this responsiveness with the simple HTTP protocol and the XMLHttpRequest object...

This is the strength of WebSockets: the server running on ESP32 can act on the clients on its own initiative, without the clients having requested anything.

To convince yourself of this, take for example this tutorial on RNT:

[ESP32/ESP8266: Control Outputs with Web Server and a Physical Button Simultaneously][rnt-tuto]

Interactions between the client and the ESP32 are **unidirectional**. In fact, a procedure is activated at regular intervals to ask the ESP32 about the status of the LED:

> **HTTP GET Request to Update State (JavaScript)**  
> To keep the output state updated on the web server, we call the following function that makes a new request on the `/state` URL every second.
>
```js
setInterval(function ( ) {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
      var inputChecked;
      var outputStateM;
      if( this.responseText == 1){ 
        inputChecked = true;
        outputStateM = "On";
      }
      else { 
        inputChecked = false;
        outputStateM = "Off";
      }
      document.getElementById("output").checked = inputChecked;
      document.getElementById("outputState").innerHTML = outputStateM;
    }
  };
  xhttp.open("GET", "/state", true);
  xhttp.send();
}, 1000 ) ;
```
>
> <div class="video-wrapper shadow with-caption" style="width:360px;height:202px;">
>  <video class="video" autoplay muted loop>
>   <source src="{{ 'videos/rnt-remote-control.mp4' | relative_url }}" type="video/mp4" />
>   Your browser does not support the video tag.
>  </video>
> </div>
>
> Note the update latency
> {: .caption }

The client updates itself, but not at the initiative of the server, at the moment the event occurs! It updates at a frequency of 1 Hz, thanks to the `setInterval` function of Javascript, because **it continuously asks the server for its new state**. This process generates a multitude of HTTP streams, with an exchange of information every second. These continuous exchanges are of no interest and continuously clog up the network for nothing.

[Parts and Wiring]({{ '/parts-and-wiring/' | relative_url }}){: .btn .btn-purple }
{: .navigator }



[rnt]:      https://randomnerdtutorials.com/
[rnt-tuto]: https://randomnerdtutorials.com/esp32-esp8266-web-server-physical-button/
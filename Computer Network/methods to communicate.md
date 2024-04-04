## Preface

For modern real-time web application,the ability to send events from the server to the client is indispensable. This necessity has led to the development of several methods over the years, each with its own set of advantages and drawbacks. Initially,long-polling was the only option available. It was then succeeded by Websockets,which offered a more robust solution for bidrectional communication.
Following WebSockets, Server-Sent Events(SSE) provided a simpler method for one-way communication from server to client. Looking ahead,the WebTransport protocol promises to revolutionize this landscape even further by providing a more efficent,flexible,and scalable approach.For niche use cases,WebRTC might also be considered for server-client events.

This article aims to delve into these technologies,comparing their performance,highlighting their benefits and limitations, and offering recommendations for various use cases to help developers make informed decisions when building real-time web applications.

## What is Long Polling

Long polling was the first "hack" to enable a server-client messaging method that can be used in browsers over HTTP. The technique emulates server push communications with normal XHR requests. Unlike traditional polling, where the client repeatedly requests data from the server at regular intervals,long polling establishes a connection to the server taht remains open until new data is available. Once the server has new information, it sends the response to the client, and the connection is closed. Immediately after receiving the server's response, the client initiates a new request, and the process repeats.This method allows for more immediate data updates and reduces unnecessary network traffic and server load.However, it can still introduce delays in communication and is less efficient than other real-time technologies like WebSockets.
```javascript
// long-polling in a JavaScript client
function longPoll() {
    fetch('http://example.com/poll')
        .then(response => response.json())
        .then(data => {
            console.log("Received data:", data);
            longPoll(); // Immediately establish a new long polling request
        })
        .catch(error => {
            /**
             * Errors can appear in normal conditions when a 
             * connection timeout is reached or when the client goes offline.
             * On errors we just restart the polling after some delay.
             */
            setTimeout(longPoll, 10000);
        });
}
longPoll(); // Initiate the long polling
```
Implementing long-polling on the client side is pretty simple, as shown in the code above. However on the backend there can be multiple difficulties to ensure the client receives all events and does not miss out updates when the client is currently reconnecting.

## What are WebSockets

WebSockets provide a full-duplex communication channel over a single, long-lived connection between the client and server. This technology enables browsers and servers to exchange data without the overhead of HTTP request-response cycles, facilitating real-time data transfer for applications like live chat,gaming,or financial trading platforms. WebSockets represent a significant advancement over traditional HTTP by allowing both parties to send data independently once the connection is established,making it ideal for scenarios that require low latency and high-frequency updates.
```javascript
// WebSocket in a JavaScript client
const socket = new WebSocket('ws://example.com');

socket.onopen = function(event) {
  console.log('Connection established');
  // Sending a message to the server
  socket.send('Hello Server!');
};

socket.onmessage = function(event) {
  console.log('Message from server:', event.data);
};
```
While the basics of the WebSocket API are easy to use it has shown to be rather complex in production. A socket can loose connection and must be re-created accordingly. Especially detecting if a connection is still usable or not can be very tricky.Mostly you would add a ping-and pong heartbeat to ensure that the open connection is not closed. This complexity is why most people use a library on top of WebSockets like Socket.IO which handles all these cases and even provides fallbacks to long-polling if required.

## What are Server-Sent-Events

Server-Sent Events(SSE) provide a standard way to push server updates to the client over HTTP. Unlike WebSockets, SSEs are disigned exclusively for one-way communication from server to client, making them ideal for scenarios like live news feeds,sports scores,or any situation where the client needs to be updated in real time without sending data to the server.

You can think of Server-Sent-Events as a single HTTP request where the backend does not send the whole body at onece,but instead keeps the connection open and trickles the answer by sending a single line each time an event has to be send to the client.

Creating a connection for receiving events with SSE is straightforward.On the client side in a browser,you initialize an EventSource instance with the URL of the server-side script that generates the events.

Listening for messages involves attaching event handlers directly to the EventSource instance. The API distinguishes between generic message events and named events, allowing for more structured communication. Here's how you can set it up in javascript.
```javascript
// Connecting to the server-side event stream
const evtSource = new EventSource("https://example.com/events");

// Handling generic message events
evtSource.onmessage = event => {
    console.log('got message: ' + event.data);
};
```
In difference to WebSockets, an EventSource will automatically reconnect on connection loss.

On the server side,your script must set the Content-Type header to text/event-stream and format each message according to the SSE specification. This includes specifying event types, data payloads, and optional fields like event ID and retry timing.

Here's how you can set up a simple SSE endpoint in a Node.js Express app:
```javascript
import express from 'express';
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/events', (req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive',
    });

    const sendEvent = (data) => {
        // all message lines must be prefixed with 'data: '
        const formattedData = `data: ${JSON.stringify(data)}\n\n`;
        res.write(formattedData);
    };

    // Send an event every 2 seconds
    const intervalId = setInterval(() => {
        const message = {
            time: new Date().toTimeString(),
            message: 'Hello from the server!',
        };
        sendEvent(message);
    }, 2000);

    // Clean up when the connection is closed
    req.on('close', () => {
        clearInterval(intervalId);
        res.end();
    });
});
app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
```

## What is the WebTransport API

WebTransport is a cutting-edge API designed for efficient, low-latency communication between web clients and servers. It leverages the [HTTP/3 QUIC protocol](https://en.wikipedia.org/wiki/HTTP/3) to enable a variety of data transfer capabilities, such as sending data over multiple streams, in both reliable and unreliable manners, and even allowing data to be sent out of order. This makes WebTransport a powerful tool for applications requiring high-performance networking,such as real-time gaming,live streaming, and collaborative platforms.However, it's important to note that WebTransport is currently a working draft and has not yet achieved widespread adoption.As of March 2024, WebTransport is in a working draft and not widely supported.You cannot yet use WebTransport in the Safari browser and there is also no native support in Node.js. This limits its usability across different platforms and environments.

Even when WebTransport will become widely supported, its API is very complex to use and likely it would be something where people build libraries on top WebTransport, not using it directly in an application's sourcecode.

## What is WebRTC

[WebRTC](https://webrtc.org/?hl=zh-cn)(Web Real-Time Communication) is an open-source project and API standard that enables real-time communication capabilities directly within web browsers and mobile applications without the need for complex server infrastructure or the installation of additional plugins.It supports peer-to-peer connections for steaming audio,video, and data exchange between browsers.WebRTC is a designed to work through NATs and firewalls,utilizing protocols like ICE,STUN,and TURN to establish a connection between peers.

While WebRTC is made to be used for client-client interactions,it could alse be leveraged for server-client communication where the server just simulated being also a client. This approach only makes sense for niche cases which is why in the following WebRTC will be ignored as an option.

The problem is that for WebRTC to work,you need a signaling-server anyway which would then again run over websockets, SSE or WebTransport. This defeats the purpose of using WebRTC as a replacement for these technologies.

## Limitations of the technologies




---
title: Mastering Node.js - book review
date: 2014-08-20
categories: books
---

## Overview

I'm really impressed by Node.js (and JavaScript) ecosystem. I took _Mastering Node.js_ book to understand Node.js philosophy and compare to JVM world.

## V8

JavaScript virtual machine, conceptually very similar to JVM. 
The most important element of JavaScript ecosystem if you want to do something more than client side web application.
I really like Node.js REPL, experimentation is as easy as with Scala.

## Event loop

Elegant simulation of concurrency. Do you remember Swing event dispatch thread and `invokeLater()` method? Event loop is the same.
It is crucial to understand events handling order:

* emitted event
* timers
* IO callbacks
* deferred execution blocks

## Event driven concurrency

Process is a first class citizen. The easiest (and cheapest) way to achieve concurrency with horizontal scalability. 

## Real-time applications

I enhanced drawing board presented in the book. It was great fun together with my 2 years old son :-)
Scalable server side implementation is presented below, I could not even imagine Java version. 

``` javascript
var express = require('express')
var path = require('path');
var app = express();
var http = require('http').Server(app);
var io = require('socket.io')(http);
var redis = require('socket.io-redis');

io.adapter(redis({ host: 'localhost', port: 6379 }));

var port = parseInt(process.argv[2]);

app.use(express.static(path.join(__dirname, 'assets')));

app.get('/', function(req, res){
  res.sendfile('index.html');
});

io.on('connection', function (socket) {
  socket.on('move', function (data) {
    socket.broadcast.emit('moving', data);
  });
});

http.listen(port, function(){
  console.log('Board started on: ' + port);
});
```

Keep in mind that SSE is unidirectional from server to clients and requires funny 2KB padding. I didn't know that before.

## Scaling on single node

Spawning, forking child processes is easy, communication between parent and children processes is easy as well.
Cluster module simplifies web application implementation for multi-core processors and it is very easy to understand and control.

## Horizontal scaling

Keep shared state in horizontally scalable store, e.g: session data in Redis or RabbitMq for events.

## Apache Bench

Command line tool for load/stress testing. Using JMeter or Gatling is not always the only way to perform simple test.

## UDP / Multicast

Good to know the world behind HTTP/REST/SOAP ... There is a lot of important layers between application and wire, do you remember OSI? 

## AWS

I have to practice using S3 or DynamoDB eventually.

## Node debugger

OMG - I used to debug application using console 10 years ago or so ;-)

## Express, Socket.io, Path 

Implementing web application using Node.js only is feasible but with Express it is much easier.

Be aware that there are thousands of web frameworks for Node.js on the market. Much more that for Java 10 years ago ;-)
It seems that frameworks built around WebSocket and Single Page App should be the leaders.

## Interesing resources

[Comparing the Performance of Web Server Architectures](https://cs.uwaterloo.ca/~brecht/papers/getpaper.php?file=eurosys-2007.pdf)

[Broken Promises](http://www.futurealoof.com/posts/broken-promises.html)

## Summary

JavaScript and Node.js seem to be one of the most vital ecosystem for web development. 
The adoption in the enterprise world is still low but I really like this ecosystem and its community.
And I'm still waiting for final version of ES6, sometimes JavaScript really sucks.

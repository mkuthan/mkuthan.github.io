---
title: "Building Your Ultimate Homelab -- part 1"
date: 2024-06-30
tags: [DIY, Homelab]
---

Welcome, fellow tech enthusiast! 🚀 If you’re anything like me, you’ve probably dreamed of having your own little corner of the digital universe — a place where wires hum, servers whir, and blinking LEDs create a symphony of possibilities. Well, my friend, you’re about to embark on an exciting adventure as we dive headfirst into the world of homelabs.

## Why a Homelab?

Before we delve into the nitty-gritty details of hardware, software, and network configurations, let’s take a moment to ponder why homelabs matter. Imagine having a playground where you can experiment, learn, and tinker without any constraints. Whether you’re a seasoned sysadmin, a budding developer, or just someone who loves gadgets, a homelab provides the canvas for your digital masterpiece.

### The Hardware Chronicles

Our journey begins with the tangible, the hardware that forms the backbone of your homelab. Picture this: a sleek gateway guarding the entrance, a managed switch orchestrating data flows, access points spreading Wi-Fi magic, a sturdy rack shelf cradling your servers, and a trusty UPS ensuring uptime even during power outages. Oh, and let’s not forget the ZigBee — because who doesn’t love a touch of automation?

In this first installment, we’ll explore each piece of hardware, demystifying their roles and unraveling the magic behind their blinking lights. From choosing the right server to optimizing cable management, we’ve got you covered.

### Stay Tuned

So, grab your favorite mug of coffee (or tea, if that’s your jam), find a cozy spot, and join me as we venture into the heart of homelab creation. In the next post, we’ll fire up our Proxmox hypervisor, spin up some virtual machines, and explore the software side of things. Spoiler alert: Home Assistant is waiting in the wings!

Remember, this isn’t just about hardware — it’s about building a playground where curiosity reigns supreme. Buckle up, my friend; the homelab adventure awaits! 🌟

## Vision and Initial Requirements

When I started building my first homelab, I needed some vision and initial requirements to avoid costly mistakes. I meticulously researched hardware options, software configurations, and scalability considerations. As I laid the foundation for my homelab, I envisioned a versatile environment that would serve both my personal projects and professional development. This foresight allowed me to make informed decisions, optimize my setup, and create a robust infrastructure that continues to evolve.

### Network

* Centrally managed network devices to avoid manual and repetitive tasks.
* Dedicated network devices, "backbone" network must work if server is down.
* [Category 6 ethernet cables](https://en.wikipedia.org/wiki/Category_6_cable), to allow easily upgrades up to [10GBASE-T](https://en.wikipedia.org/wiki/10_Gigabit_Ethernet#10GBASE-T).
* [Gigabit ethernet](https://en.wikipedia.org/wiki/Gigabit_Ethernet#1000BASE-T) network devices to strike a balance between performance and affordability.
* [Wi-Fi 802.11ax](https://en.wikipedia.org/wiki/Wi-Fi_6) to utilize Gigabit network. [Mesh technology](https://en.wikipedia.org/wiki/Wireless_mesh_network) to extend coverage by creating a seamless network across my home and surroundings.
* [Power over Ethernet](https://en.wikipedia.org/wiki/Power_over_Ethernet) to simplify the deployment of Wi-Fi access points, IP cameras, and other network devices. With PoE, you can transmit both data and power over a single Ethernet cable, eliminating the need for separate power sources.

### Server

* Power efficient to keep electricity costs under control.
* Multi-core CPU with high clock speeds and memory enough to handle all my virtual machines.
* GPU for hardware accelerated vision inference.
* NVMe SSD disk for OS hosting to get lightning-fast read and write speeds.
* Enterprise grade SATA disk for virtual machines to achieve reliability, durability, consistent performance, and avoid quick disk wear.
* USB3 to connect external HDD storage for camera recordings and backups.

### Cameras

* IP cameras with [RTSP](https://en.wikipedia.org/wiki/Real-Time_Streaming_Protocol) for wide compatibility with various recording and monitoring systems.
* Can operate without internet access. The last thing I would want is for the cameras to have access to the internet.
* Decent image quality. Fixed length, 8Mpx/4Mpx with at least 1/3 CMOS image sensor.
* IR illumination to get night vision.
* H.265 codec to get better video quality at given bitrate compared to older codes like H.264.
* Minimal “Smart” features. Rely on open-source software for recording and detection, and avoid paying extra for features like facial recognition or cloud-based analytics.

### IoT

* [ZigBee](https://en.wikipedia.org/wiki/Zigbee), to avoid vendor specific solutions
* Wi-Fi devices only if there are no ZigBee alternatives, they have to work in local network without cloud access and integrate with Home Assistant.

### Other parts

* 19'' 9U rack as organized space for mounting your network equipment, servers, and other devices.
* Patch panel for neatly organized Ethernet cables. It allows me to connect devices to the network without messy cable runs.
* Small UPS (Uninterruptible Power Supply) to ensure uninterrupted power supply during short electricity outages or fluctuations. Compatible with with open-source software for monitoring and control.

## Big picture

Below, you can observe the current state of the network equipment, servers, and other devices neatly mounted in the utility room, positioned just above the door leading to the garage.

![Rack](/assets/images/2024-06-30-homlab-hardware/rack.jpg)

Given that all the cables are well organized, understanding the entire topology can still be challenging. Below, you’ll find the logical connection diagram for all the devices:

```mermaid
flowchart LR
    modem[Modem 5G] == eth PoE === poe[PoE injector]
    poe -- eth --- router[Router]
    router -- eth --- switch[Managed Switch]

    switch -- eth --- server[Server]
    switch == eth PoE === aps[Access Points]
    switch == eth PoE === cameras[Cameras]
    switch -- eth --- heatpump[Heat Pump]
    switch -- eth --- tv[TV Box]
    switch -- eth --- computers_eth[Computers]

    server -- usb --- zigbee[ZigBee Gate]
    server -- usb --- hdds[External HDDs]
    server -- usb --- ups[UPS]

    aps -. wifi .- phones[Phones]
    aps -. wifi .- computers_wifi[Computers]
    aps -. wifi .- iot_wifi[IoT Devices]

    zigbee -. zigbee .- iot_zigbee[IoT Devices]
```

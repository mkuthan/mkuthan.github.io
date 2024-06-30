---
title: "Building Your Ultimate Homelab -- hardware"
date: 2024-06-30
tags: [DIY, Homelab]
header:
    overlay_image: /assets/images/2024-06-30-homelab-hardware/overlay.jpg
    caption: ""
---

Welcome, fellow tech enthusiast!
If you’re anything like me, you’ve probably dreamed of having your own little corner of the digital universe—a place where wires hum, servers whir, and blinking LEDs create a symphony of possibilities.
Well, my friend, you’re about to embark on an exciting adventure as we dive headfirst into the world of homelabs.

## Why a Homelab

Before we delve into the nitty-gritty details of hardware, software, and network configurations, let’s take a moment to ponder why homelabs matter.
Imagine having a playground where you can experiment, learn, and tinker without any constraints.
Whether you’re a seasoned sysadmin, a budding developer, or just someone who loves gadgets, a homelab provides the canvas for your digital masterpiece.

## The Hardware Chronicles

Our journey begins with the tangible, the hardware that forms your homelab.
Picture this: a sleek gateway guarding the entrance, a managed switch orchestrating data flows, access points spreading Wi-Fi magic, a sturdy rack shelf cradling your servers, and a trusty UPS ensuring uptime even during power outages.
Oh, and let’s not forget the Zigbee—because who doesn’t love a touch of automation?

In this first installment, we’ll explore each piece of hardware, demystifying their roles and unraveling the magic behind their blinking lights.
From choosing the right server to optimizing cable management, we’ve got you covered.

## Vision and Initial Requirements

When I started building my first homelab, I needed some vision and initial requirements to avoid costly mistakes.
I meticulously researched hardware options, software configurations, and scalability considerations.
As I laid the foundation for my homelab, I envisioned a versatile environment that would serve both my personal projects and professional development.
This foresight allowed me to make informed decisions, optimize my setup, and create a robust infrastructure that continues to evolve.

### Network

* Centrally managed network devices to avoid manual and repetitive tasks.
* Dedicated network devices, networks must operate if the server is down.
* [Category 6 Ethernet cables](https://en.wikipedia.org/wiki/Category_6_cable), to enable future upgrades up to [10GBASE-T](https://en.wikipedia.org/wiki/10_Gigabit_Ethernet#10GBASE-T).
* [Gigabit Ethernet](https://en.wikipedia.org/wiki/Gigabit_Ethernet#1000BASE-T) network devices to strike a balance between performance and affordability.
* [Wi-Fi 802.11ax](https://en.wikipedia.org/wiki/Wi-Fi_6) to take advantage of the Gigabit network.
* [Mesh Wi-Fi technology](https://en.wikipedia.org/wiki/Wireless_mesh_network) to extend coverage by creating a seamless network across my home and surroundings.
* [Power over Ethernet](https://en.wikipedia.org/wiki/Power_over_Ethernet) to simplify the deployment of Wi-Fi access points, IP cameras, and other network devices.

### Server

* Power efficient to keep electricity costs under control.
* Multi-core CPU with high clock speeds and enough memory to handle all my virtual machines.
* GPU for hardware accelerated vision inference.
* NVMe SSD disk for OS to get lightning-fast read and write speeds.
* Enterprise grade SATA disk for virtual machines to achieve reliability, durability, consistent performance, and avoid quick disk wear.
* USB3 ports to connect external HDD storage for camera recordings and backups.

### Cameras

* IP cameras with [RTSP](https://en.wikipedia.org/wiki/Real-Time_Streaming_Protocol) for wide compatibility with various recording and monitoring systems.
* Can operate without internet access. The last thing I would want is for the cameras to have access to the internet.
* Decent image quality. Fixed focus length, 8Mpx/4Mpx with at least 1/3'' CMOS image sensor.
* IR illumination to get night vision.
* H.265 codec to get better video quality at given bit rate compared to older codes like H.264.
* Avoid vendor specific "smart" features. Rely on open-source software for recording and detection, and avoid paying extra for features like facial recognition or cloud-based analytics.

### IoT

* Don't over-engineer, smart homes should enhance the user experience, not complicate it. Essential functionalities must operate if the home automation is down.
* Base on [Zigbee](https://en.wikipedia.org/wiki/Zigbee) mesh network, to avoid vendor specific Wi-Fi solutions.
* Wi-Fi devices only if there are no Zigbee viable options. They have to work in a local network without cloud access and integrate well with Home Assistant.

### Other Parts

* 19'' rack for mounting your network equipment, servers, and other devices.
* Patch panel for neatly organized Ethernet cables.
* Small UPS to ensure uninterrupted power supply during short electricity outages or fluctuations. Compatible with open-source software for monitoring and control.

## Big Picture

I initiated the setup of my Homelab by meticulously planning the computer network in the house, but the end result resembled a tangle of cables emerging from the walls in the utility room.

![Cables](/assets/images/2024-06-30-homelab-hardware/cables.jpg)

It took considerable effort and time to build everything from scratch. Now, below, you can observe the current state of the network equipment, servers, and other devices—neatly mounted and strategically positioned just above the door leading to the garage.

![Rack](/assets/images/2024-06-30-homelab-hardware/rack.jpg)

Given that all the cables are well organized, understanding the entire topology can still be challenging. Below, you’ll find the physical Homelab connection diagram, devices mounted in the rack cabinet are marked orange:

```mermaid
flowchart LR
    modem[Radio Modem] == eth PoE === poe[PoE injector]:::rack
    poe -- eth --- router[Router]:::rack
    router -- eth --- switch[Managed Switch]:::rack

    switch -- eth --- server[Server]:::rack
    switch == eth PoE === aps[Access Points]
    switch == eth PoE === cameras[Cameras]
    switch -- eth --- iot_lan[IoT Devices]
    switch -- eth --- tv[TV Box]
    switch -- eth --- computers_eth[Computers]

    server -- usb --- zigbee[Zigbee Gate]
    server -- usb --- hdds[External HDDs]:::rack
    server -- usb --- ups[UPS]:::rack

    aps -. wifi .- phones[Phones]
    aps -. wifi .- computers_wifi[Computers]
    aps -. wifi .- iot_wifi[IoT Devices]

    zigbee -. zigbee .- iot_zigbee[IoT Devices]

    classDef rack fill:Orange
```

## Radio Modem

Given the absence of optical fiber at my homelab installation site, I rely on radio access for internet connectivity. My service provider has installed the [Ubiquiti airMAX LiteBeam 5AC](https://eu.store.ui.com/eu/en/pro/products/litebeam-5ac), an ultra-lightweight outdoor wireless station specifically designed for point-to-point communication. The base station is situated a little over 2 kilometers away, and the reported latency on my WAN link is approximately 22 milliseconds.

![Ubiquiti airMAX LiteBeam 5AC](/assets/images/2024-06-30-homelab-hardware/modem.jpg)

## TP-Link Omada Network Devices

I made the deliberate choice to deploy TP-Link network devices from their business line, expertly managed by the Omada controller.
Notably more budget-friendly than the alternatives offered by Ubiquiti Unifi, these TP-Link devices seamlessly meet all my networking requirements.

### ER605 Router

Gigabit router [ER605](https://www.tp-link.com/en/business-networking/vpn-router/er605/) is a straightforward and functional model that provides essential features without unnecessary frills.

![TP-Link ER605 Router](/assets/images/2024-06-30-homelab-hardware/er605.jpg)

### SG2428P Managed Switch

28-Port Gigabit switch [TL-SG2428P](https://www.tp-link.com/en/business-networking/omada-switch-poe/tl-sg2428p/v1/) is a robust managed switch equipped with PoE and VLAN support.

![TP-Link SG2428P Switch](/assets/images/2024-06-30-homelab-hardware/sg2428P.jpg)

### EAP610 Access Points

WiFi 6 access points [EAP610](https://www.tp-link.com/en/business-networking/omada-wifi-ceiling-mount/eap610/v3/) with simultaneous 574 Mbps on 2.4 GHz and 1201 Mbps on 5 GHz speeds.

![TP-Link EAP610 Access Point](/assets/images/2024-06-30-homelab-hardware/eap610.jpg)

## Dell Optiplex 3050 Server

A few years ago, I embarked on a project to create a 24/7 home server using a [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/).
While the Pi served its purpose, my evolving Homelab demanded more horsepower without compromising energy efficiency.
After careful consideration, I opted for a used [Dell Optiplex](https://en.wikipedia.org/wiki/Dell_OptiPlex) system equipped with an Intel i5 7th generation CPU.
The compact small form factor housing allowed me to maximize space while still achieving the desired performance.

![Dell Optiplex 3050 Server](/assets/images/2024-06-30-homelab-hardware/optiplex3050.webp)

* CPU Intel i5-7500T 2.7GHz, 4 cores
* GPU Intel® HD Graphics 630
* 32GB RAM DDR4 2666MHz
* PCI Express 3.0
* Built-in Gigabit Ethernet card
* USB 3.0 × 4, 2.0 × 2
* Samsung PM981 256 GB (NVMe, TLC)

### Intel DC S3610 SSD

When it comes to hosting virtual machines (VMs), using enterprise-grade SSDs over consumer grade SSDs offers several advantages: endurance, reliability, consistent performance and power loss protection.
I installed a used Intel DC S3610 800 GB (SATA, MLC) SSD with over 50,000 power-on hours, and remarkably, it still reports only 2% wearout.

![Intel DC S3610 SSD](/assets/images/2024-06-30-homelab-hardware/s3610.jpg)

### RTL8125B Network Card

For desktop computers, a single Gigabit Ethernet card suffices.
However, when setting up a server, having at least two network devices becomes crucial. Fortunately, since I didn’t require WiFi or Bluetooth functionality, I replaced the stock Intel Dual-Band Wireless-AC 8265 card with a PCIe M.2 A+E 2.5GB RTL8125B card.

![RTL8125B Network Card](/assets/images/2024-06-30-homelab-hardware/rtl8125b.jpg)

## Eaton 5S UPS

Here’s why I chose the Eaton 5S 1000VA UPS for my homelab?

* My homelab has an average power consumption of 75---80 Watts.
* Its line-interactive technology adjusts input voltage fluctuations without switching to battery power unless necessary.
* During outages, it ensures uninterrupted operation for approximately 30 minutes.
* It fits neatly into a 19’’ network rack.
* Network UPS Tools (NUT) on Linux systems simplifies monitoring and management.

![Eaton 5S UPS](/assets/images/2024-06-30-homelab-hardware/5s1000.jpg)

## Dahua IP Cameras

Given my concerns about the quality of no-name Chinese products, I deliberately opted for the reputable brand Dahua.
While I also evaluated Hikvision, the affordability factor tipped the scales in favor of Dahua. Specifically, Dahua’s 8 Mpx cameras ([IPC-HFW2841S](https://www.dahuasecurity.com/products/All-Products/Network-Cameras/WizSense-Series/2-Series/8MP/IPC-HFW2841S-S)) come in at approximately $100, while their 4 Mpx counterparts ([IPC-HFW2441S](https://www.dahuasecurity.com/products/All-Products/Network-Cameras/WizSense-Series/2-Series/4MP/IPC-HFW2441S-S)) are priced around $70.

![Dahua IP Camera](/assets/images/2024-06-30-homelab-hardware/dahua.jpg)

### Storage Requirements

To get decent H.265 video quality at 5 FPS I set up: 6Mb/s for 8Mpx cameras and 3Mb/s for 4Mpx cameras.
How much storage do I need to store 1 month of recording from `2 x 8 Mpx` and `6 x 4 Mpx` cameras?

Hourly rate per camera:

```
8Mpx: 6 [Mb/s] * 3600 seconds * 1/8 =  2.64 [GiB]
4Mpx: 3 [Mb/s] * 3600 seconds * 1/8 = 1.32 [GiB]
```

Monthly rate for all cameras:

```
2 * 2.64 [GiB] * 24 hours * 30 days = 3802 [GiB]
6 * 1.32 [GiB] * 24 hours * 30 days = 5702 [GiB]
```

Total required capacity:

```
3802 [GiB] + 5702 [GiB] = 9.5 TiB
```

### WD Purple HDD

Recordings from my surveillance system are securely stored on a dedicated [WD Purple Pro 12TB](https://www.westerndigital.com/products/internal-drives/wd-purple-pro-sata-hdd?sku=WD121PURP) hard drive. This robust drive supports an impressive workload rate of up to 550TB per year, achieving speeds of up to 245MB/s. During write operations, the average power consumption remains at a modest 6.6 Watts.

![WD Purple HDD](/assets/images/2024-06-30-homelab-hardware/wd-purple.jpg)

The hard drive is connected to the server using a USB3 to SATA adapter with [UASP](https://en.wikipedia.org/wiki/USB_Attached_SCSI) support.
Additionally, for 3.5-inch disks, a 12V/2A power adapter is required.

![USB3 to SATA adapter](/assets/images/2024-06-30-homelab-hardware/usb3sata.jpg)

## Sonoff ZBDongle-E

Here’s why I chose the Sonoff ZBDongle-E for my homelab?

* Sonoff ZBDongle-E seamlessly integrates with Home Assistant’s integration, making it an excellent choice for Zigbee networks within my smart home.
* It's based on the Silicon Labs EFR32MG21 SoC (System-on-Chip).
This powerful chip ensures reliable communication and efficient performance.
* To extend the signal range, I connected the Sonoff ZBDongle-E using a 1.5m USB extension cable and mounted it outside the 19’’ rack.

![Sonoff ZBDongle-E](/assets/images/2024-06-30-homelab-hardware/efr32mg21.jpg)

## Stay Tuned

In the next post, we’ll fire up our Proxmox hypervisor, spin up some virtual machines, and explore the software side of things.
Spoiler alert: Home Assistant is waiting in the wings!

![Proxmox hypervisor](/assets/images/2024-06-30-homelab-hardware/proxmox.png)

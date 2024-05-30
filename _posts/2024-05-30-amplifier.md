---
title: DIY hi-fi stereo system
date: 2024-05-30
tags: [hardware, DIY]
header:
    overlay_image: /assets/images/2024-05-30-amplifier/overlay.jpg
    caption: ""
---

This blog post is different from all other posts I've already published.
I swapped my keyboard for a soldering iron and IDE for a circuit board to build a DIY hi-fi stereo system.

## Parts list

* High quality 2-way bookshelf loudspeakers
* Multimedia streaming box
* [Class D](https://en.wikipedia.org/wiki/Class-D_amplifier) digital power amplifier board
* Optical [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (Digital Analog Converter) board
* Step down power supply board
* 3D printed housing

## Bookshelf speakers

My requirements:

* Bookshelf speakers with modern and minimalist design
* Decent sound quality
* Sized to my living room -- 35 m²
* Budget $300

Trusted expert recommended [JBL Stage A130](https://www.jbl.com/loudspeakers/STAGE+A130.html),
2 way, 5.25" woofer, 1" aluminum tweeter bookshelf loudspeakers.
I visited local home audio store, listen for a while to get my own opinion and … put the box with the speakers to the car - they sound amazing 😀

![JBL Stage A130](/assets/images/2024-05-30-amplifier/jbl-a130.jpg)

## Multimedia streaming box

My requirements:

* Optical `S/PDIF` output
* Integration with Spotify Connect and other streaming platforms
* Easy to use for my family members
* Power efficient
* Budget $50

I found a used TV box with Android 11 based on the powerful [Amlogic](https://en.wikipedia.org/wiki/Amlogic) S905X4 SoC.

![TV box](/assets/images/2024-05-30-amplifier/dv8988.jpg)

## Class D digital power amplifier board

My requirements:

* Based on opinionated Class-D audio amplifier chip
* With enough power to drive JBL Stage A130 speakers
* Well made using high quality capacitors and coils

I decided for amplifier board based on [TPA3255](https://www.ti.com/lit/ds/symlink/tpa3255.pdf?ts=1716361845588&ref_url=https%253A%252F%252Fwww.google.com%252F) chip from [Aliexpress](https://www.aliexpress.com/item/1005004883731361.html), model [BDM8-A](https://files.sure-electronics.com/download/BDM8A.pdf).

![Amplifier board](/assets/images/2024-05-30-amplifier/bdm8-a.jpg)

## Optical DAC decoder board

My requirements:

* Based on opinionated DAC chip
* Optical `S/PDIF` input

I decided for DAC board based on [CS8416](https://www.cirrus.com/products/cs8416/) and [CS4398](https://www.cirrus.com/products/cs4398/) chips from [Aliexpress](https://www.aliexpress.com/item/32960837911.html) with up to 24bits at 192kHz sampling.

![Digital analog converter board](/assets/images/2024-05-30-amplifier/dac.jpg)

## Step down power supply board

My requirements:

* With enough power to drive `TPA3255` amplifier and 6Ω speakers
* Power efficient
* With low noise ripple
* Small and light

Some math to estimate maximum power available with 24V DC power supply and 6Ω speakers:

1. 24V DC => 24V / √2 ~= 17V AC
2. 17V^2 / 6Ω = 48W

48W per channel looks fine for selected speakers and size of my living room, lets calculate current consumption:

1. 17V / 6Ω = 2.83A
2. 2.83A * 2 (channels) = 5.66A

I decided for power supply board from [Aliexpress](https://www.aliexpress.com/item/1005005916644628.html), version: 24V/6A.

![Power supply board](/assets/images/2024-05-30-amplifier/psu.jpg)

## Other parts

* Transformer 2×12V 0.08A for DAC
* Universal circuit board
* Power socket
* Circuit breakers
* Connectors and cables
* Nylon screws, nuts and spacers
* PLA filament for 3D print

## Assembly

Component layout to get low profile housing and all connectors on the back side:

![Amplifier - layout](/assets/images/2024-05-30-amplifier/amplifier-layout.jpg)

Working prototype in the cardboard box:

![Amplifier - prototype](/assets/images/2024-05-30-amplifier/amplifier-prototype.jpg)

Assembly on a thin furniture board:

![Amplifier - plate](/assets/images/2024-05-30-amplifier/amplifier-plate.jpg)

## Housing

My 3D printer has a maximum print volume of 18×18×18 cm, so I printed the housing in two parts.

Left side 3D model:

![3D - left](/assets/images/2024-05-30-amplifier/3d-left.png)

Right side 3D model:

![3D - right](/assets/images/2024-05-30-amplifier/3d-right.png)

After just 14 hours of printing, I could make the first adjustments:

![Amplifier - case](/assets/images/2024-05-30-amplifier/amplifier-case.jpg)

## Final result

![Amplifier - final](/assets/images/2024-05-30-amplifier/amplifier-final.jpg)

If I had to sum up why it's worth playing DIY:

1. **Educational Aspect:** Constructing a multimedia stereo system from parts is an excellent way to learn about electronics, audio system design, Android TV and 3D printing.

2. **Satisfaction:** There’s a sense of accomplishment in creating something functional and beautiful. I was most proud that the first 3D printed housing fit like a glove.

3. **Customization:** When you build your own amplifier, you have complete control over the design. You can tailor it to your preferences, for example I didn't want any hardware knobs.

4. **Cost Savings:** DIY amplifiers can be more affordable than commercial high-end options but you can't count your time.

I would also like to thank Jacek for his inspiration and help.
